# Go Reflect Approach for SDK v2 to Plugin Framework Bridge

## Context

After determining that the wrapper pattern is blocked (SDK v2 data sources can't be called from Plugin Framework), we're exploring whether Go's `reflect` package could provide a runtime bridge between the two frameworks.

---

## What is Go Reflect?

Go's `reflect` package enables **runtime type inspection and manipulation**. It allows programs to:

1. **Inspect types at runtime** - Determine the type of an `interface{}` value
2. **Access struct fields dynamically** - Get/set field values by name
3. **Call methods dynamically** - Invoke methods on objects at runtime
4. **Create instances dynamically** - Instantiate types at runtime
5. **Convert between types** - Transform values between compatible types

### Key Concepts

```go
import "reflect"

// Get the Type of a value
t := reflect.TypeOf(myVar)     // Returns reflect.Type

// Get the Value (runtime representation)
v := reflect.ValueOf(myVar)    // Returns reflect.Value

// Access struct fields
field := v.FieldByName("Name") // Returns reflect.Value of field

// Set values (requires pointer and settable value)
v.Elem().FieldByName("Name").SetString("new value")

// Check the Kind (int, struct, slice, etc.)
if v.Kind() == reflect.Struct { ... }
```

### References
- [Official reflect package docs](https://pkg.go.dev/reflect)
- [Sling Academy Tutorial](https://www.slingacademy.com/article/using-the-reflect-package-for-runtime-type-inspection-in-go/)
- [Relia Software Guide](https://reliasoftware.com/blog/reflection-in-golang)

---

## Current Type Systems

### SDK v2 (`terraform-plugin-sdk/v2`)

```go
// Schema definition
map[string]*schema.Schema{
    "name": {
        Type:     schema.TypeString,
        Required: true,
    },
    "enabled": {
        Type:     schema.TypeBool,
        Computed: true,
    },
}

// Data access via ResourceData
value := d.Get("name").(string)
d.Set("enabled", true)
```

### Plugin Framework (`terraform-plugin-framework`)

```go
// Schema definition
schema.Schema{
    Attributes: map[string]schema.Attribute{
        "name": schema.StringAttribute{
            Required: true,
        },
        "enabled": schema.BoolAttribute{
            Computed: true,
        },
    },
}

// Data access via typed model
type Model struct {
    Name    types.String `tfsdk:"name"`
    Enabled types.Bool   `tfsdk:"enabled"`
}
```

---

## Key Discovery: Resource.Data() Method

The SDK v2 `*schema.Resource` has a `Data()` method that creates `*ResourceData` at runtime:

```go
// From terraform-plugin-sdk/v2/helper/schema
func (r *Resource) Data(s *terraform.InstanceState) *ResourceData
```

### Real Usage in Provider

Found in `mmv1/third_party/terraform/services/resourcemanager/list_google_service_account.go:92`:

```go
// Creates ResourceData from a resource schema
d := ResourceGoogleServiceAccount().Data(&terraform.InstanceState{})

// Populate with values
if err := d.Set("project", project); err != nil {
    return fmt.Errorf("error setting project: %w", err)
}

// Use for API calls
url, err := tpgresource.ReplaceVars(d, config, "{{IAMBasePath}}projects/{{project}}/serviceAccounts")
```

**This is the key insight**: We can create `*ResourceData` at runtime without needing the full Terraform execution context.

---

## Proposed Reflect-Based Bridge Pattern

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REFLECT-BASED BRIDGE PATTERN                             │
└─────────────────────────────────────────────────────────────────────────────┘

  Plugin Framework Ephemeral              Bridge Layer                SDK v2 Data Source
  ┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐
  │                         │     │                         │     │                         │
  │  Open(req, resp) {      │     │  BridgeRead(pfModel,    │     │  dataSourceXxxRead(     │
  │    model := Model{}     │────▶│              dsSchema,  │────▶│      d, meta)           │
  │    req.Config.Get(model)│     │              readFunc)  │     │                         │
  │                         │     │                         │     │  d.Get("field")         │
  │    // Call bridge       │     │  1. Create ResourceData │     │  d.Set("field", val)    │
  │    bridgeResult :=      │     │  2. Populate from model │     │                         │
  │      BridgeRead(...)    │     │  3. Call SDK v2 Read    │     └─────────────────────────┘
  │                         │     │  4. Extract values      │                │
  │    // Convert result    │◀────│  5. Return as map       │◀───────────────┘
  │    model.Field =        │     │                         │
  │      types.StringValue  │     └─────────────────────────┘
  │                         │
  │    resp.Result.Set(model│
  │  }                      │
  └─────────────────────────┘
```

### Implementation Concept

```go
package bridge

import (
    "reflect"

    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
    "github.com/hashicorp/terraform-plugin-sdk/v2/terraform"
    "github.com/hashicorp/terraform-plugin-framework/types"
)

// BridgeResult contains data extracted from SDK v2 ResourceData
type BridgeResult struct {
    Values map[string]interface{}
    ID     string
    Error  error
}

// BridgeRead calls an SDK v2 Read function and extracts the results
func BridgeRead(
    inputValues map[string]interface{},
    sdkResource *schema.Resource,
    meta interface{},
) BridgeResult {

    // 1. Create ResourceData from the schema
    d := sdkResource.Data(&terraform.InstanceState{})

    // 2. Populate ResourceData with input values using reflect
    for key, value := range inputValues {
        if err := d.Set(key, value); err != nil {
            return BridgeResult{Error: err}
        }
    }

    // 3. Call the SDK v2 Read function
    err := sdkResource.Read(d, meta)
    if err != nil {
        return BridgeResult{Error: err}
    }

    // 4. Extract all values from ResourceData using reflect
    result := make(map[string]interface{})
    for key := range sdkResource.Schema {
        result[key] = d.Get(key)
    }

    return BridgeResult{
        Values: result,
        ID:     d.Id(),
    }
}

// ConvertToPluginFramework converts bridge result to Plugin Framework types
func ConvertToPluginFramework(result BridgeResult, model interface{}) error {
    v := reflect.ValueOf(model).Elem()
    t := v.Type()

    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        fieldType := t.Field(i)

        // Get the tfsdk tag to find the attribute name
        tfsdkTag := fieldType.Tag.Get("tfsdk")
        if tfsdkTag == "" {
            continue
        }

        // Get the value from the bridge result
        value, exists := result.Values[tfsdkTag]
        if !exists {
            continue
        }

        // Convert to Plugin Framework type based on field type
        switch field.Type() {
        case reflect.TypeOf(types.String{}):
            if strVal, ok := value.(string); ok {
                field.Set(reflect.ValueOf(types.StringValue(strVal)))
            }
        case reflect.TypeOf(types.Bool{}):
            if boolVal, ok := value.(bool); ok {
                field.Set(reflect.ValueOf(types.BoolValue(boolVal)))
            }
        case reflect.TypeOf(types.Int64{}):
            if intVal, ok := value.(int); ok {
                field.Set(reflect.ValueOf(types.Int64Value(int64(intVal))))
            }
        // ... handle other types
        }
    }

    return nil
}
```

### Usage in Ephemeral Resource

```go
func (e *ephemeralSecretManagerSecretVersion) Open(ctx context.Context, req ephemeral.OpenRequest, resp *ephemeral.OpenResponse) {
    var model ephemeralSecretManagerSecretVersionModel
    resp.Diagnostics.Append(req.Config.Get(ctx, &model)...)

    // Convert Plugin Framework model to plain map
    inputValues := map[string]interface{}{
        "project":              model.Project.ValueString(),
        "secret":               model.Secret.ValueString(),
        "version":              model.Version.ValueString(),
        "is_secret_data_base64": model.IsSecretDataBase64.ValueBool(),
    }

    // Call bridge to SDK v2 data source
    result := bridge.BridgeRead(
        inputValues,
        DataSourceSecretManagerSecretVersion(), // SDK v2 data source schema
        e.providerConfig,                       // meta
    )

    if result.Error != nil {
        resp.Diagnostics.AddError("Bridge error", result.Error.Error())
        return
    }

    // Convert result back to Plugin Framework model
    if err := bridge.ConvertToPluginFramework(result, &model); err != nil {
        resp.Diagnostics.AddError("Conversion error", err.Error())
        return
    }

    resp.Diagnostics.Append(resp.Result.Set(ctx, model)...)
}
```

---

## What Reflect Enables

### 1. Dynamic Schema Conversion

Reflect can read SDK v2 schema and generate Plugin Framework schema:

```go
func ConvertSchema(sdkSchema map[string]*schema.Schema) map[string]schema.Attribute {
    result := make(map[string]schema.Attribute)

    for name, s := range sdkSchema {
        switch s.Type {
        case schema.TypeString:
            result[name] = schema.StringAttribute{
                Required:  s.Required,
                Optional:  s.Optional,
                Computed:  s.Computed,
                Sensitive: s.Sensitive,
            }
        case schema.TypeBool:
            result[name] = schema.BoolAttribute{
                Required: s.Required,
                Optional: s.Optional,
                Computed: s.Computed,
            }
        // ... other types
        }
    }

    return result
}
```

### 2. Dynamic Model Population

Reflect can set fields on a Plugin Framework model dynamically:

```go
func PopulateModel(model interface{}, values map[string]interface{}) {
    v := reflect.ValueOf(model).Elem()
    t := v.Type()

    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        tag := t.Field(i).Tag.Get("tfsdk")

        if val, ok := values[tag]; ok {
            // Use reflect to set the appropriate types.X value
            setPluginFrameworkValue(field, val)
        }
    }
}
```

### 3. Dynamic ResourceData Access

Reflect can extract all values set on ResourceData:

```go
func ExtractAllValues(d *schema.ResourceData, sdkSchema map[string]*schema.Schema) map[string]interface{} {
    result := make(map[string]interface{})
    for key := range sdkSchema {
        result[key] = d.Get(key)
    }
    return result
}
```

---

## Existing Reflect Usage in Codebase

### Example 1: Dynamic Field Access (`fwtransport/framework_utils.go:360`)

**This is the most relevant example** - it shows dynamic field access by name:

```go
// In BuildReplacementFunc - replaces {{FieldName}} with config values
// 'm' is a string variable containing the field name

if f := reflect.Indirect(reflect.ValueOf(config)).FieldByName(m); f.IsValid() {
    return f.String()
}
```

This dynamically accesses a field on `*transport_tpg.Config` using a string variable. Without reflect, you'd need a switch statement for every possible field name.

**How we'd use this pattern:**
```go
// Dynamically get Plugin Framework model field value
fieldName := "Project"  // from tfsdk tag
field := reflect.Indirect(reflect.ValueOf(&model)).FieldByName(fieldName)
method := field.MethodByName("ValueString")
value := method.Call(nil)[0].String()  // "my-project"
```

### Example 2: Type Conversion (`tpgresource/convert.go`)

Used for converting between API response types:

```go
func Convert(item, out interface{}) error {
    bytes, err := json.Marshal(item)
    if err != nil {
        return err
    }
    err = json.Unmarshal(bytes, out)
    if err != nil {
        return err
    }
    // Using reflect to handle omitted fields
    if _, ok := item.(map[string]interface{}); !ok {
        setOmittedFields(item, out)
    }
    return nil
}

func setOmittedFields(item, out interface{}) {
    iVal := reflect.ValueOf(item).Elem()
    oVal := reflect.ValueOf(out).Elem()

    for i := 0; i < iVal.NumField(); i++ {
        // ... uses reflect to copy fields between structs
    }
}
```

---

## Key Challenges and Limitations

### 1. Context/Timeout Handling

SDK v2 Read functions don't use `context.Context`:
```go
// SDK v2
func dataSourceXxxRead(d *schema.ResourceData, meta interface{}) error

// Plugin Framework
func (e *ephemeral) Open(ctx context.Context, req, resp)
```

**Mitigation**: The underlying transport uses context internally; this may not be a blocker.

### 2. Diagnostics vs Error

SDK v2 returns `error`, Plugin Framework uses `diag.Diagnostics`:
```go
// SDK v2
return fmt.Errorf("error reading: %s", err)

// Plugin Framework
resp.Diagnostics.AddError("Error reading", err.Error())
```

**Mitigation**: Easy to convert `error` to `Diagnostics.AddError()`.

### 3. Complex Nested Types

Nested objects, lists, and maps require recursive conversion:
```go
// SDK v2 nested
"labels": {
    Type: schema.TypeMap,
    Elem: &schema.Schema{Type: schema.TypeString},
}

// Plugin Framework nested
"labels": schema.MapAttribute{
    ElementType: types.StringType,
}
```

**Mitigation**: Build comprehensive type mapping with recursion.

### 4. Performance Overhead

Reflect operations are slower than direct type access:
- ~10-100x slower for simple operations
- Should cache reflection results where possible

**Mitigation**:
- Cache `reflect.Type` lookups
- Only use reflect for bridge operations, not hot paths

### 5. Null vs Unknown vs Zero Values

Plugin Framework distinguishes between null, unknown, and zero values:
```go
types.StringNull()    // Null value
types.StringUnknown() // Unknown (computed during apply)
types.StringValue("") // Zero value
```

SDK v2 doesn't have this distinction.

**Mitigation**: Use `GetOk()` to detect if value was set:
```go
if v, ok := d.GetOk("field"); ok {
    // Value was explicitly set
    model.Field = types.StringValue(v.(string))
} else {
    // Value not set
    model.Field = types.StringNull()
}
```

### 6. State/Config Separation

SDK v2 ResourceData mixes config and state, Plugin Framework separates them:
```go
// Plugin Framework
req.Config.Get(ctx, &configModel)  // Configuration only
req.State.Get(ctx, &stateModel)    // State only
```

**Mitigation**: For Read operations (data sources/ephemeral), this is less of an issue since we only read config and return results.

---

## Feasibility Assessment

| Aspect | Feasibility | Notes |
|--------|-------------|-------|
| Create ResourceData at runtime | ✅ **Proven** | `Resource.Data()` method exists and is used |
| Populate ResourceData | ✅ **Proven** | `d.Set()` works at runtime |
| Call SDK v2 Read function | ✅ **Feasible** | Just a function call with `d` and `meta` |
| Extract values from ResourceData | ✅ **Feasible** | `d.Get()` for each schema field |
| Convert to Plugin Framework types | ✅ **Feasible** | Reflect can set typed fields |
| Handle nested types | ⚠️ **Complex** | Requires recursive conversion |
| Handle all edge cases | ⚠️ **Unknown** | Need testing |

### Verdict: **POTENTIALLY FEASIBLE**

The reflect-based bridge pattern appears technically feasible. The main questions are:

1. **Completeness**: Can we handle all type combinations?
2. **Edge cases**: What happens with complex nested structures?
3. **Maintenance**: Is this more maintainable than standalone generation?

---

## Comparison with Standalone Generation

| Aspect | Reflect Bridge | Standalone Generation |
|--------|----------------|----------------------|
| Code reuse | ✅ Reuses SDK v2 logic | ❌ Duplicates API logic |
| Maintenance | ⚠️ Bridge code complexity | ⚠️ Keep ephemeral in sync |
| Performance | ⚠️ Reflect overhead | ✅ Direct calls |
| Type safety | ⚠️ Runtime type checking | ✅ Compile-time safety |
| Implementation effort | Medium | Medium |
| Risk | Medium (edge cases) | Low (proven pattern) |

---

## Recommended Next Steps

1. **Prototype**: Build a minimal bridge for one simple data source
2. **Test**: Verify it works for basic types (string, bool, int)
3. **Expand**: Add support for complex types (list, map, nested)
4. **Benchmark**: Measure performance impact
5. **Decide**: Compare with standalone generation effort

---

## Prototype Scope

Start with `google_secret_manager_secret_version`:
- Simple schema (strings, bools)
- No complex nested types
- Has both data source and ephemeral versions to compare
- Well-understood behavior

If the prototype succeeds, expand to more complex resources.
