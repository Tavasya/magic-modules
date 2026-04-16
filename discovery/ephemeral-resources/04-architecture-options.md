# Architecture Options for Unified Generation

## Overview

This document explores different architectural approaches for generating both data sources and ephemeral resources from a unified definition.

## Option A: Wrapper Pattern (Recommended)

### Concept

Generate ephemeral resources that wrap the underlying data source logic. The ephemeral resource calls a shared "core" function that performs the actual data fetching.

### Implementation

1. **Generate a shared core function** that performs the API calls
2. **Data source** uses SDK v2 and calls the core function
3. **Ephemeral resource** uses Plugin Framework and calls the same core function

### YAML Configuration

```yaml
name: 'SecretVersion'
description: 'A secret version resource'
base_url: 'projects/{{project}}/secrets/{{secret}}/versions'
id_format: 'projects/{{project}}/secrets/{{secret}}/versions/{{version}}'

datasource_experimental:
  generate: true
  generate_ephemeral: true  # NEW: Generate ephemeral resource too
  sensitive_fields:         # NEW: Fields to mark as sensitive
    - secret_data

properties:
  - name: 'secret'
    type: String
    required: true
  - name: 'version'
    type: String
    required: false
  - name: 'secretData'
    type: String
    output: true
```

### Generated Files

```
google/services/secretmanager/
├── secret_version_core.go           # Shared logic (NEW)
├── data_source_secret_version.go    # SDK v2 data source
└── ephemeral_secret_version.go      # Plugin Framework ephemeral (NEW)
```

### Pros
- Clean separation of concerns
- Easy to maintain shared logic
- Explicit sensitive field marking
- Both SDK v2 and Plugin Framework supported

### Cons
- Requires new "core" template
- Slightly more complex generation

---

## Option B: Ephemeral as Data Source Adapter

### Concept

Generate ephemeral resources that internally instantiate and call the data source read function, converting types between SDK v2 and Plugin Framework.

### Implementation

1. **Data source** is generated as-is (SDK v2)
2. **Ephemeral resource** wraps the data source, converting types

### Generated Code Pattern

```go
// ephemeral_secret_version.go
func (e *ephemeralSecretVersion) Open(ctx context.Context, req ephemeral.OpenRequest, resp *ephemeral.OpenResponse) {
    // Create a fake schema.ResourceData from Plugin Framework types
    d := createResourceDataFromFrameworkModel(req.Config)

    // Call the data source read function
    err := dataSourceSecretVersionRead(d, e.providerConfig)

    // Convert back to Plugin Framework types
    model := convertResourceDataToModel(d)
    resp.Result.Set(ctx, model)
}
```

### Pros
- Maximum code reuse
- Data source is the "source of truth"
- Minimal template changes

### Cons
- Complex type conversion at runtime
- Performance overhead
- Fragile coupling between frameworks
- Hard to debug

---

## Option C: Dual Generation from Same Definition

### Concept

Generate both data sources and ephemeral resources directly from the same YAML definition, with separate templates that produce framework-appropriate code.

### Implementation

1. **Single YAML definition** describes the data source
2. **Two templates** generate SDK v2 and Plugin Framework code
3. **Shared helper functions** for common logic

### YAML Configuration

```yaml
name: 'SecretVersion'
datasource_experimental:
  generate: true

ephemeral_experimental:       # NEW: Separate block for ephemeral
  generate: true
  sensitive_fields:
    - secret_data
```

### Templates

```
mmv1/templates/terraform/
├── datasource.go.tmpl                # Existing SDK v2 template
├── ephemeral.go.tmpl                 # NEW: Plugin Framework template
└── ephemeral_test.go.tmpl            # NEW: Test template
```

### Pros
- Clean separation of templates
- Full control over generated code
- Independent evolution possible

### Cons
- More code duplication in templates
- Need to maintain two parallel templates
- Schema definitions duplicated

---

## Option D: Plugin Framework Only

### Concept

Migrate data sources to Plugin Framework and share code directly with ephemeral resources.

### Implementation

1. **Convert data sources** to Plugin Framework
2. **Share schema definitions** between data sources and ephemeral resources
3. **Single generation path** for both

### Generated Code Pattern

```go
// Shared schema
var secretVersionSchema = schema.Schema{
    Attributes: map[string]schema.Attribute{
        "secret": schema.StringAttribute{Required: true},
        // ...
    },
}

// Data source uses shared schema
func (d *dataSourceSecretVersion) Schema(...) {
    resp.Schema = secretVersionSchema
}

// Ephemeral uses same schema with sensitive marked
func (e *ephemeralSecretVersion) Schema(...) {
    s := secretVersionSchema
    s.Attributes["secret_data"].(schema.StringAttribute).Sensitive = true
    resp.Schema = s
}
```

### Pros
- True code sharing
- Single framework
- Future-proof (Plugin Framework is the future)

### Cons
- Massive migration effort for existing data sources
- Breaking change for SDK v2 data sources
- Not all features may be available in Plugin Framework yet

---

## Option E: Code Generation with Shared API Layer

### Concept

Create a shared API layer that both data sources and ephemeral resources use, with thin wrappers for each framework.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    YAML Definition                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Shared API Layer                           │
│  - API client calls                                         │
│  - Data transformation                                      │
│  - Error handling                                           │
└─────────────────────────────────────────────────────────────┘
                     │                    │
                     ▼                    ▼
        ┌─────────────────┐    ┌────────────────────┐
        │  SDK v2 Wrapper │    │ Plugin FW Wrapper  │
        │  (Data Source)  │    │ (Ephemeral)        │
        └─────────────────┘    └────────────────────┘
```

### Pros
- Clean architecture
- API logic is framework-agnostic
- Easy to test API layer independently

### Cons
- Requires significant refactoring
- More generated code
- Harder to inject custom code

---

## Recommendation: Option A (Wrapper Pattern)

### Rationale

1. **Incremental adoption** - Can be added to existing data sources
2. **Clear separation** - Each framework has its own code
3. **Maintainable** - Shared logic is explicit
4. **YAML-driven** - Configuration is declarative
5. **Testable** - Each component can be tested independently

### Implementation Phases

**Phase 1: Infrastructure**
- Add `ephemeral_experimental` YAML block
- Create `ephemeral.go.tmpl` template
- Add `GenerateEphemeralResource()` to terraform.go
- Update `framework_provider.go.tmpl` to include generated ephemeral resources

**Phase 2: Core Logic Extraction**
- Create pattern for extracting shared API logic
- Generate `*_core.go` files for shared functions

**Phase 3: Migration**
- Convert existing handwritten ephemeral resources to generated
- Deprecate handwritten versions

### Estimated Complexity

| Component | Effort | Risk |
|-----------|--------|------|
| YAML schema changes | Low | Low |
| New template creation | Medium | Medium |
| Provider registration | Medium | Low |
| Core logic extraction | High | Medium |
| Testing infrastructure | Medium | Low |

### Success Criteria

1. Can generate `ephemeral_google_secret_manager_secret_version` from YAML
2. Generated code passes existing tests
3. No regression in data source functionality
4. Clear documentation for YAML configuration
