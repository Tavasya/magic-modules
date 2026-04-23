# Go Reflect for Beginners: How It Applies to Our Problem

## The Problem We're Trying to Solve

We have two different "worlds" in the Terraform provider:

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│         SDK v2 World            │     │     Plugin Framework World      │
│    (Old way, data sources)      │     │    (New way, ephemeral)         │
├─────────────────────────────────┤     ├─────────────────────────────────┤
│                                 │     │                                 │
│  Types:                         │     │  Types:                         │
│    schema.TypeString            │     │    types.String                 │
│    schema.TypeBool              │     │    types.Bool                   │
│    schema.TypeInt               │     │    types.Int64                  │
│                                 │     │                                 │
│  Data access:                   │     │  Data access:                   │
│    d.Get("field")               │     │    model.Field.ValueString()    │
│    d.Set("field", value)        │     │    model.Field = types.StringValue(x) │
│                                 │     │                                 │
│  Returns: error                 │     │  Returns: Diagnostics           │
│                                 │     │                                 │
└─────────────────────────────────┘     └─────────────────────────────────┘
              │                                        │
              │         CAN'T TALK DIRECTLY            │
              └────────────────X───────────────────────┘
```

**The challenge**: Ephemeral resources (Plugin Framework) want to reuse data source logic (SDK v2), but they speak different "languages."

---

## What is Go Reflect?

### Normal Go: You Know Types at Compile Time

Normally in Go, you know exactly what type you're working with:

```go
// You know it's a string
name := "hello"
fmt.Println(name)

// You know the struct has a Name field
type Person struct {
    Name string
    Age  int
}

p := Person{Name: "Alice", Age: 30}
fmt.Println(p.Name)  // You type "Name" directly
```

### Reflect: You Discover Types at Runtime

With reflect, you can inspect and manipulate types **without knowing them ahead of time**:

```go
import "reflect"

// I have "something" but I don't know what it is
func inspectAnything(something interface{}) {
    // What type is it?
    t := reflect.TypeOf(something)
    fmt.Println("Type:", t)  // e.g., "main.Person"

    // What's inside it?
    v := reflect.ValueOf(something)
    fmt.Println("Value:", v)  // e.g., "{Alice 30}"

    // If it's a struct, what fields does it have?
    if t.Kind() == reflect.Struct {
        for i := 0; i < t.NumField(); i++ {
            field := t.Field(i)
            fmt.Println("Field:", field.Name, "Type:", field.Type)
        }
    }
}

// Usage:
inspectAnything(Person{Name: "Alice", Age: 30})
// Output:
// Type: main.Person
// Value: {Alice 30}
// Field: Name Type: string
// Field: Age Type: int
```

### Reflect Can Also CHANGE Values

```go
func setFieldByName(obj interface{}, fieldName string, newValue interface{}) {
    // Get a "reflection" of the object (must pass pointer to modify)
    v := reflect.ValueOf(obj).Elem()

    // Find the field by name
    field := v.FieldByName(fieldName)

    // Set the new value
    if field.Kind() == reflect.String {
        field.SetString(newValue.(string))
    }
}

// Usage:
p := Person{Name: "Alice", Age: 30}
setFieldByName(&p, "Name", "Bob")
fmt.Println(p.Name)  // "Bob"
```

---

## Why Reflect Helps Our Problem

### The Core Insight

Both SDK v2 and Plugin Framework ultimately work with the same basic Go types underneath:
- Strings are still `string`
- Booleans are still `bool`
- Integers are still `int`

They just **wrap** them differently:

```go
// SDK v2: stores values in a map
d.Get("name")  // returns interface{}, you cast it: d.Get("name").(string)

// Plugin Framework: stores values in typed structs
model.Name.ValueString()  // returns string
```

### Reflect Lets Us Translate Between Them

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         REFLECT AS TRANSLATOR                           │
└─────────────────────────────────────────────────────────────────────────┘

  Plugin Framework                 Reflect                    SDK v2
  ┌──────────────────┐     ┌───────────────────────┐     ┌──────────────────┐
  │                  │     │                       │     │                  │
  │  model.Name      │────▶│  Extract string value │────▶│  d.Set("name",   │
  │  (types.String)  │     │  "hello"              │     │         "hello") │
  │                  │     │                       │     │                  │
  └──────────────────┘     └───────────────────────┘     └──────────────────┘

                                    ...SDK v2 does its work...

  ┌──────────────────┐     ┌───────────────────────┐     ┌──────────────────┐
  │                  │     │                       │     │                  │
  │  model.Name =    │◀────│  Wrap string value    │◀────│  d.Get("name")   │
  │  types.StringVal │     │  "result"             │     │  returns "result"│
  │                  │     │                       │     │                  │
  └──────────────────┘     └───────────────────────┘     └──────────────────┘
```

---

## Step-by-Step: How It Would Work

### Step 1: Start with Plugin Framework Input

User configures an ephemeral resource:

```hcl
ephemeral "google_secret_manager_secret_version" "my_secret" {
  project = "my-project"
  secret  = "my-secret"
  version = "latest"
}
```

This becomes a Plugin Framework model:

```go
type Model struct {
    Project types.String `tfsdk:"project"`
    Secret  types.String `tfsdk:"secret"`
    Version types.String `tfsdk:"version"`
    // ... output fields
}

// Terraform populates it:
model := Model{
    Project: types.StringValue("my-project"),
    Secret:  types.StringValue("my-secret"),
    Version: types.StringValue("latest"),
}
```

### Step 2: Use Reflect to Extract Plain Values

```go
// Convert Plugin Framework model to plain Go map
func extractValues(model interface{}) map[string]interface{} {
    result := make(map[string]interface{})

    v := reflect.ValueOf(model)
    t := v.Type()

    // Loop through all fields
    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        fieldInfo := t.Field(i)

        // Get the tfsdk tag (e.g., `tfsdk:"project"`)
        tfsdkName := fieldInfo.Tag.Get("tfsdk")

        // Extract the actual value based on type
        switch field.Type().String() {
        case "types.String":
            // Call ValueString() method via reflect
            method := field.MethodByName("ValueString")
            strValue := method.Call(nil)[0].String()
            result[tfsdkName] = strValue

        case "types.Bool":
            method := field.MethodByName("ValueBool")
            boolValue := method.Call(nil)[0].Bool()
            result[tfsdkName] = boolValue
        }
    }

    return result
}

// Result:
// map[string]interface{}{
//     "project": "my-project",
//     "secret":  "my-secret",
//     "version": "latest",
// }
```

### Step 3: Create SDK v2 ResourceData and Populate It

```go
// Get the data source schema
dataSource := DataSourceSecretManagerSecretVersion()

// Create a ResourceData instance (this is the key discovery!)
d := dataSource.Data(&terraform.InstanceState{})

// Populate it with our extracted values
for key, value := range extractedValues {
    d.Set(key, value)
}
```

### Step 4: Call the SDK v2 Read Function

```go
// This is the existing data source Read function
// It makes API calls, processes data, and calls d.Set() for results
err := dataSourceSecretManagerSecretVersionRead(d, providerConfig)
```

### Step 5: Extract Results from ResourceData

```go
// Get all the values that were set by the Read function
results := make(map[string]interface{})
for fieldName := range dataSource.Schema {
    results[fieldName] = d.Get(fieldName)
}

// Result:
// map[string]interface{}{
//     "project":     "my-project",
//     "secret":      "my-secret",
//     "version":     "5",
//     "secret_data": "super-secret-value",
//     "create_time": "2024-01-15T...",
//     "enabled":     true,
// }
```

### Step 6: Use Reflect to Populate Plugin Framework Model

```go
func populateModel(model interface{}, values map[string]interface{}) {
    v := reflect.ValueOf(model).Elem()  // Need Elem() because model is a pointer
    t := v.Type()

    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        fieldInfo := t.Field(i)

        tfsdkName := fieldInfo.Tag.Get("tfsdk")
        value, exists := values[tfsdkName]
        if !exists {
            continue
        }

        // Set the field based on its type
        switch field.Type().String() {
        case "types.String":
            if strVal, ok := value.(string); ok {
                // Create types.StringValue and set it
                newVal := reflect.ValueOf(types.StringValue(strVal))
                field.Set(newVal)
            }

        case "types.Bool":
            if boolVal, ok := value.(bool); ok {
                newVal := reflect.ValueOf(types.BoolValue(boolVal))
                field.Set(newVal)
            }
        }
    }
}
```

### Step 7: Return the Result

```go
// Now model has all the values from the SDK v2 data source!
// Set it as the ephemeral resource result
resp.Result.Set(ctx, model)
```

---

## The Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           COMPLETE BRIDGE FLOW                              │
└─────────────────────────────────────────────────────────────────────────────┘

  TERRAFORM USER                    OUR CODE                         GCP API
       │                               │                                │
       │  ephemeral "google_secret..." │                                │
       │  {                            │                                │
       │    project = "my-proj"        │                                │
       │    secret  = "my-secret"      │                                │
       │  }                            │                                │
       │                               │                                │
       ▼                               │                                │
  ┌─────────────────────┐              │                                │
  │ Plugin Framework    │              │                                │
  │ creates Model with  │              │                                │
  │ types.String values │              │                                │
  └─────────┬───────────┘              │                                │
            │                          │                                │
            ▼                          │                                │
  ┌─────────────────────┐              │                                │
  │ REFLECT: Extract    │              │                                │
  │ plain Go values     │              │                                │
  │ "my-proj", "my-sec" │              │                                │
  └─────────┬───────────┘              │                                │
            │                          │                                │
            ▼                          │                                │
  ┌─────────────────────┐              │                                │
  │ Create SDK v2       │              │                                │
  │ ResourceData        │              │                                │
  │ d.Set("project",..) │              │                                │
  └─────────┬───────────┘              │                                │
            │                          │                                │
            ▼                          │                                │
  ┌─────────────────────┐              │                                │
  │ Call SDK v2 Read    │──────────────┼───────────────────────────────▶│
  │ dataSourceXxxRead() │              │          GET /secrets/...      │
  │                     │◀─────────────┼────────────────────────────────│
  │ d.Set("secret_data")│              │          { data: "..." }       │
  └─────────┬───────────┘              │                                │
            │                          │                                │
            ▼                          │                                │
  ┌─────────────────────┐              │                                │
  │ Extract results     │              │                                │
  │ from ResourceData   │              │                                │
  │ d.Get("secret_data")│              │                                │
  └─────────┬───────────┘              │                                │
            │                          │                                │
            ▼                          │                                │
  ┌─────────────────────┐              │                                │
  │ REFLECT: Convert    │              │                                │
  │ back to types.String│              │                                │
  │ model.SecretData =  │              │                                │
  │ types.StringValue() │              │                                │
  └─────────┬───────────┘              │                                │
            │                          │                                │
            ▼                          │                                │
  ┌─────────────────────┐              │                                │
  │ Return to Terraform │              │                                │
  │ resp.Result.Set()   │              │                                │
  └─────────────────────┘              │                                │
```

---

## Simple Analogy

Think of it like **language translation**:

```
English Speaker          Translator           French Speaker
(Plugin Framework)       (Reflect)            (SDK v2)
      │                      │                     │
      │  "Hello, how        │                     │
      │   are you?"         │                     │
      │─────────────────────▶                     │
      │                      │  "Bonjour,         │
      │                      │   comment          │
      │                      │   allez-vous?"     │
      │                      │────────────────────▶
      │                      │                     │
      │                      │◀────────────────────│
      │                      │  "Je vais bien,    │
      │                      │   merci!"          │
      │                      │                     │
      │◀─────────────────────│                     │
      │  "I'm doing well,   │                     │
      │   thanks!"          │                     │
```

- **Plugin Framework** speaks "types.String"
- **SDK v2** speaks "schema.TypeString" and "d.Get()"
- **Reflect** translates between them by looking at the underlying values

---

## Why This Might Work

1. **We can create ResourceData at runtime**
   - Found real usage: `d := Resource.Data(&terraform.InstanceState{})`
   - This is already done in the codebase

2. **ResourceData has simple Get/Set methods**
   - `d.Set("field", value)` takes any value
   - `d.Get("field")` returns the value

3. **Plugin Framework types have accessor methods**
   - `types.String` has `ValueString()` that returns plain `string`
   - `types.StringValue(s)` creates a `types.String` from plain `string`

4. **Reflect can call methods and set fields dynamically**
   - We don't need to hardcode every field
   - Reflect discovers them at runtime

---

## What Could Go Wrong

| Challenge | Explanation |
|-----------|-------------|
| **Complex nested types** | Lists, maps, nested objects need recursive handling |
| **Null vs empty** | Plugin Framework distinguishes `null` from `""`, SDK v2 doesn't |
| **Performance** | Reflect is slower (but probably OK for our use case) |
| **Edge cases** | Some type combinations might not convert cleanly |

---

## Summary

**Reflect is like X-ray vision for Go types.** Instead of knowing types at compile time, you discover and manipulate them at runtime.

For our problem:
1. Take Plugin Framework model → use reflect to get plain values
2. Put plain values into SDK v2 ResourceData
3. Call SDK v2 Read function (reuses existing code!)
4. Get plain values back from ResourceData
5. Use reflect to put them into Plugin Framework model

**The benefit**: We reuse all the existing SDK v2 data source logic instead of duplicating it.

**The risk**: Complex types and edge cases might cause problems.

**Next step**: Build a prototype to see if it actually works.
