# Technical Considerations

## Framework Differences

### SDK v2 vs Plugin Framework

| Feature | SDK v2 | Plugin Framework |
|---------|--------|------------------|
| **Context** | Implicit | Explicit `context.Context` |
| **Types** | Go primitives | `types.String`, `types.Bool`, etc. |
| **Null handling** | Empty values | `types.StringNull()` |
| **Unknown handling** | Not supported | `types.StringUnknown()` |
| **Schema definition** | `map[string]*schema.Schema` | `schema.Schema{Attributes: ...}` |
| **Read function** | `Read(d, meta) error` | `Read(ctx, req, resp)` |
| **Diagnostics** | `error` return | `resp.Diagnostics.Append()` |

### Type Mapping

```go
// SDK v2 → Plugin Framework
schema.TypeString  → types.StringType / types.String
schema.TypeInt     → types.Int64Type / types.Int64
schema.TypeBool    → types.BoolType / types.Bool
schema.TypeList    → types.ListType / types.List
schema.TypeSet     → types.SetType / types.Set
schema.TypeMap     → types.MapType / types.Map
```

## Schema Generation Challenges

### 1. Nested Objects

**SDK v2:**
```go
"nested_block": {
    Type: schema.TypeList,
    Elem: &schema.Resource{
        Schema: map[string]*schema.Schema{
            "field": {Type: schema.TypeString},
        },
    },
}
```

**Plugin Framework:**
```go
"nested_block": schema.ListNestedAttribute{
    NestedObject: schema.NestedAttributeObject{
        Attributes: map[string]schema.Attribute{
            "field": schema.StringAttribute{},
        },
    },
}
```

### 2. Validators

**SDK v2:**
```go
"field": {
    Type:         schema.TypeString,
    ValidateFunc: validation.StringInSlice([]string{"A", "B"}, false),
}
```

**Plugin Framework:**
```go
"field": schema.StringAttribute{
    Validators: []validator.String{
        stringvalidator.OneOf("A", "B"),
    },
}
```

### 3. Default Values

**SDK v2:**
```go
"field": {
    Type:    schema.TypeString,
    Default: "default_value",
}
```

**Plugin Framework:**
```go
"field": schema.StringAttribute{
    // No direct default, use planmodifier
    PlanModifiers: []planmodifier.String{
        stringplanmodifier.UseStateForUnknown(),
    },
}
// Or handle in Open() method
```

## Sensitive Field Detection

### Automatic Detection

Fields that should be marked as `Sensitive: true`:
- Fields with names containing: `password`, `secret`, `token`, `key`, `credential`
- Fields with `sensitive: true` in YAML
- Fields that contain authentication data

### YAML Configuration

```yaml
properties:
  - name: 'secretData'
    type: String
    sensitive: true  # Explicit marking
```

### Template Logic

```go
{{- if or .Sensitive (containsSensitivePattern .Name) }}
Sensitive: true,
{{- end }}
```

## Provider Registration

### Current SDK v2 Registration

```go
// Via registry system
func init() {
    registry.Schema{
        Name:        "google_secret_manager_secret_version",
        ProductName: "secretmanager",
        Type:        registry.SchemaTypeDataSource,
        Schema:      DataSourceSecretManagerSecretVersion(),
    }.Register()
}
```

### Plugin Framework Registration

```go
// Via provider method
func (p *FrameworkProvider) EphemeralResources(_ context.Context) []func() ephemeral.EphemeralResource {
    return []func() ephemeral.EphemeralResource{
        // Handwritten
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
        // Generated (NEW)
        secretmanager.GoogleEphemeralSecretManagerSecretVersion,
    }
}
```

### Challenge: Generated Registration

Need to generate a list of ephemeral resources and include in the provider. Options:

**Option 1: Generated File**
```go
// generated_ephemeral_resources.go
var generatedEphemeralResources = []func() ephemeral.EphemeralResource{
    secretmanager.EphemeralSecretManagerSecretVersion,
    // ... more generated
}
```

Then in `framework_provider.go.tmpl`:
```go
func (p *FrameworkProvider) EphemeralResources(...) []func() ephemeral.EphemeralResource {
    result := []func() ephemeral.EphemeralResource{
        // Handwritten
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
    }
    result = append(result, generatedEphemeralResources...)
    return result
}
```

**Option 2: Template-Generated List**
```go
// framework_provider.go.tmpl
func (p *FrameworkProvider) EphemeralResources(...) []func() ephemeral.EphemeralResource {
    return []func() ephemeral.EphemeralResource{
        // Handwritten
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
        {{- range $product := $.EphemeralResources }}
        {{ $product.Package }}.Ephemeral{{ $product.Name }},
        {{- end }}
    }
}
```

## API Layer Considerations

### Transport Layer Reuse

Both SDK v2 and Plugin Framework can use the same transport layer:

```go
transport_tpg.SendRequest(transport_tpg.SendRequestOptions{
    Config:    config,
    Method:    "GET",
    Project:   project,
    RawURL:    url,
    UserAgent: userAgent,
})
```

### Config Access

**SDK v2:**
```go
config := meta.(*transport_tpg.Config)
```

**Plugin Framework:**
```go
// In Configure()
pd, _ := req.ProviderData.(*transport_tpg.Config)
p.providerConfig = pd

// In Open()
config := p.providerConfig
```

## Testing Considerations

### Ephemeral Resource Testing

Ephemeral resources are harder to test because:
1. No state to verify
2. Values may change between runs
3. Side effects are temporary

### Test Patterns

```go
func TestAccEphemeralSecretManagerSecretVersion_basic(t *testing.T) {
    t.Parallel()

    context := map[string]interface{}{
        "random_suffix": acctest.RandString(t, 10),
    }

    acctest.VcrTest(t, resource.TestCase{
        PreCheck:                 func() { acctest.AccTestPreCheck(t) },
        ProtoV5ProviderFactories: acctest.ProtoV5ProviderFactories(t),
        Steps: []resource.TestStep{
            {
                Config: testAccEphemeralSecretManagerSecretVersion_basic(context),
                // Can't use Check functions on ephemeral resources
                // Test that config is valid and doesn't error
            },
        },
    })
}

func testAccEphemeralSecretManagerSecretVersion_basic(context map[string]interface{}) string {
    return acctest.Nprintf(`
resource "google_secret_manager_secret" "secret" {
  secret_id = "tf-test-secret-%{random_suffix}"
  replication { auto {} }
}

resource "google_secret_manager_secret_version" "version" {
  secret      = google_secret_manager_secret.secret.id
  secret_data = "secret-data"
}

ephemeral "google_secret_manager_secret_version" "test" {
  secret  = google_secret_manager_secret.secret.secret_id
  version = google_secret_manager_secret_version.version.version
}

# Use the ephemeral value somewhere to ensure it's evaluated
locals {
  ephemeral_test = ephemeral.google_secret_manager_secret_version.test.secret_data
}
`, context)
}
```

## Version Compatibility

### Terraform Version Requirements

- Ephemeral resources: Terraform >= 1.10
- Provider must declare minimum Terraform version

### Provider Version

- terraform-provider-google >= 7.0 for ephemeral resources
- Need to handle version constraints in generation

## Error Handling

### SDK v2 Style

```go
if err != nil {
    return fmt.Errorf("Error reading secret: %s", err)
}
```

### Plugin Framework Style

```go
if err != nil {
    resp.Diagnostics.AddError(
        "Error reading secret",
        fmt.Sprintf("Could not read secret: %s", err),
    )
    return
}
```

## Import Statements

### Required Imports for Ephemeral Resources

```go
import (
    "context"
    "fmt"

    "github.com/hashicorp/terraform-plugin-framework/ephemeral"
    "github.com/hashicorp/terraform-plugin-framework/ephemeral/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"

    transport_tpg "github.com/hashicorp/terraform-provider-google/google/transport"
)
```

### Potential Conflicts

- Both SDK v2 and Plugin Framework have `schema` packages
- Need to alias imports carefully

## Performance Considerations

### Ephemeral Resource Lifecycle

- `Open()` is called on every plan and apply
- Need efficient API calls
- Consider caching within a single run

### Memory Usage

- Plugin Framework types have overhead
- Model structs add memory per resource instance
- Should be minimal for typical usage
