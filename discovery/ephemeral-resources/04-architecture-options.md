# Architecture Options for Unified Generation

## Overview

This document explores different architectural approaches for generating both data sources and ephemeral resources from a unified definition.

---

## ⚠️ IMPORTANT UPDATE: Blocker Identified

### Feedback from Team Discussion (with Cameron)

After reviewing the wrapper pattern approach, a **fundamental blocker** was identified:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           THE BLOCKER                                        │
└─────────────────────────────────────────────────────────────────────────────┘

  Current generated data sources work like this:

  ┌─────────────────────────────────────────────┐
  │  datasource.go.tmpl generates:              │
  │                                             │
  │  func dataSourceXxxRead(d, meta) error {    │
  │      d.SetId(id)                            │
  │      return resourceXxxRead(d, meta) ◄──────┼──── Delegates to RESOURCE
  │  }                                          │
  └─────────────────────────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────┐
  │  resourceXxxRead() is SDK v2 code!          │
  │                                             │
  │  - Uses schema.ResourceData                 │
  │  - Returns error                            │
  │  - All API logic lives here                 │
  └─────────────────────────────────────────────┘

  To wrap data source → ephemeral, we'd need:

  ┌─────────────────────────────────────────────┐
  │  Ephemeral (Plugin Framework)               │
  │                                             │
  │  func (e *ephemeral) Open() {               │
  │      // Can't call resourceXxxRead()!       │
  │      // It's SDK v2, we're Plugin Framework │
  │  }                                          │
  └─────────────────────────────────────────────┘

  THE PROBLEM:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  Ephemeral Resource ─────X────▶ Data Source ──────▶ Resource Read      │
  │  (Plugin Framework)      │      (SDK v2)            (SDK v2)            │
  │                          │                                              │
  │                     CAN'T CALL!                                         │
  │                     Different frameworks!                               │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Why the Wrapper Pattern Won't Work (For Now)

1. **Data sources delegate to resource Read** - The generated data source doesn't have its own API logic; it calls `resourceXxxRead()`

2. **Resource Read is SDK v2** - All the actual API logic (URL building, HTTP calls, response parsing) lives in the SDK v2 resource code

3. **Can't call SDK v2 from Plugin Framework** - You can't easily call `resourceXxxRead(d *schema.ResourceData, meta)` from Plugin Framework code because:
   - Different type systems (`schema.ResourceData` vs Plugin Framework model)
   - Different error handling (`error` vs `Diagnostics`)
   - Would need to create fake `schema.ResourceData` - fragile and hacky

4. **Would need full resource migration first** - To make wrapper pattern work, we'd need to migrate the underlying resource to Plugin Framework, which is a massive undertaking

### Conclusion: Wrapper Pattern is BLOCKED

The wrapper/bridge approach (Option A in this doc) **cannot be implemented** until the SDK v2 → Plugin Framework migration for resources is complete.

---

## New Direction: Standalone Ephemeral Generation

Instead of wrapping data sources, generate **standalone ephemeral resources** that have their own API logic:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NEW APPROACH: STANDALONE EPHEMERAL                        │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────┐
  │  YAML Definition                            │
  │                                             │
  │  ephemeral_experimental:                    │
  │    generate: true                           │
  └─────────────────────────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────┐
  │  Generate: ephemeral_xxx.go                 │
  │                                             │
  │  - Full Plugin Framework code               │
  │  - Own schema definition                    │
  │  - Own API logic (not shared with resource) │
  │  - Standalone, no dependencies on SDK v2    │
  └─────────────────────────────────────────────┘

  PROS:
  ✓ No dependency on SDK v2 code
  ✓ Can implement now, doesn't need migration
  ✓ Clean Plugin Framework implementation
  ✓ Independent of resource/datasource changes

  CONS:
  ✗ API logic is duplicated (ephemeral has own copy)
  ✗ Changes to API need updating in multiple places
  ✗ More generated code
```

### What This Means

| Approach | Status | Reason |
|----------|--------|--------|
| **Option A: Wrapper Pattern** | ❌ BLOCKED | Requires SDK v2 → PF migration |
| **Option B: Ephemeral as Adapter** | ❌ BLOCKED | Same reason |
| **Option C: Dual Generation (Standalone)** | ✅ VIABLE | No SDK v2 dependency |
| **Option D: Plugin Framework Only** | ❌ BLOCKED | Requires full migration |
| **Option E: Shared API Layer** | ❌ BLOCKED | Would need to refactor resources |

**Recommended Path Forward:**
1. Generate **standalone ephemeral resources** from YAML (Option C)
2. Accept the code duplication for now
3. Revisit unified generation after SDK v2 → Plugin Framework migration is complete

---

## Current State: How It Works Today

Before diving into the proposed unified approach, it's important to understand how data sources and ephemeral resources are currently implemented separately.

### Current Data Source Flow (Generated)

Data sources CAN be generated from YAML definitions using the `datasource_experimental` flag. When enabled, mmv1 generates a simple wrapper around the resource's Read function.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CURRENT DATA SOURCE GENERATION FLOW                       │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  YAML Definition (e.g., mmv1/products/cloudrun/Service.yaml)                │
│                                                                             │
│  datasource_experimental:                                                   │
│    generate: true                                                           │
│    exclude_test: true                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  mmv1/provider/terraform.go                                                 │
│                                                                             │
│  GenerateObject() {                                                         │
│      ...                                                                    │
│      t.GenerateSingularDataSource(object, templateData, outputFolder)       │
│      ...                                                                    │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Template: mmv1/templates/terraform/datasource.go.tmpl                      │
│                                                                             │
│  Key Logic:                                                                 │
│  1. Reuse resource schema: rs := Resource{{ .ResourceName }}().Schema       │
│  2. Convert to datasource: DatasourceSchemaFromResourceSchema(rs)           │
│  3. Add required fields from IdFormat                                       │
│  4. Delegate read to resource: resource{{ $.ResourceName }}Read(d, meta)    │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Generated Output: google/services/cloudrun/data_source_cloud_run_service.go│
│                                                                             │
│  func DataSourceCloudRunService() *schema.Resource {                        │
│      rs := ResourceCloudRunService().Schema  // Reuse resource schema       │
│      dsSchema := tpgresource.DatasourceSchemaFromResourceSchema(rs)         │
│      return &schema.Resource{                                               │
│          Read:   dataSourceCloudRunServiceRead,                             │
│          Schema: dsSchema,                                                  │
│      }                                                                      │
│  }                                                                          │
│                                                                             │
│  func dataSourceCloudRunServiceRead(d, meta) error {                        │
│      d.SetId(id)                                                            │
│      return resourceCloudRunServiceRead(d, meta)  // Delegate to resource   │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Registration: Via registry system (SDK v2)                                 │
│                                                                             │
│  func init() {                                                              │
│      registry.Schema{                                                       │
│          Name:        "google_cloud_run_service",                           │
│          Type:        registry.SchemaTypeDataSource,                        │
│          Schema:      DataSourceCloudRunService(),                          │
│      }.Register()                                                           │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics of Current Data Source Generation:**
- Uses **SDK v2** (`terraform-plugin-sdk/v2/helper/schema`)
- **Delegates to resource Read** function (no separate API logic)
- **Registered via registry** system
- **State is persisted** in Terraform state file
- Simple wrapper pattern - minimal code generation

---

### Current Ephemeral Resource Flow (Handwritten)

Ephemeral resources are **NOT generated** - they are entirely handwritten in `mmv1/third_party/terraform/services/`. Each one is manually coded using the Plugin Framework.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 CURRENT EPHEMERAL RESOURCE FLOW (HANDWRITTEN)                │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  NO YAML DEFINITION - Entirely handwritten                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Handwritten File:                                                          │
│  mmv1/third_party/terraform/services/secretmanager/                         │
│      ephemeral_google_secret_manager_secret_version.go                      │
│                                                                             │
│  Developer manually writes:                                                 │
│  1. Struct definition                                                       │
│  2. Model with tfsdk tags                                                   │
│  3. Metadata() method                                                       │
│  4. Schema() method                                                         │
│  5. Configure() method                                                      │
│  6. Open() method with full API logic                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Handwritten Code Structure:                                                │
│                                                                             │
│  // Must implement ephemeral.EphemeralResource interface                    │
│  var _ ephemeral.EphemeralResource = &googleEphemeralSecretManagerSV{}      │
│                                                                             │
│  type googleEphemeralSecretManagerSecretVersion struct {                    │
│      providerConfig *transport_tpg.Config                                   │
│  }                                                                          │
│                                                                             │
│  // Model with Plugin Framework types                                       │
│  type ephemeralSecretManagerSecretVersionModel struct {                     │
│      Project    types.String `tfsdk:"project"`                              │
│      Secret     types.String `tfsdk:"secret"`                               │
│      SecretData types.String `tfsdk:"secret_data"`  // Sensitive            │
│      ...                                                                    │
│  }                                                                          │
│                                                                             │
│  func (p *googleEphemeral...) Schema(...) {                                 │
│      // Manually define entire schema                                       │
│      resp.Schema = schema.Schema{                                           │
│          Attributes: map[string]schema.Attribute{                           │
│              "secret_data": schema.StringAttribute{                         │
│                  Computed:  true,                                           │
│                  Sensitive: true,  // Must manually mark                    │
│              },                                                             │
│          },                                                                 │
│      }                                                                      │
│  }                                                                          │
│                                                                             │
│  func (p *googleEphemeral...) Open(...) {                                   │
│      // Manually write ALL API call logic                                   │
│      // - Build URL                                                         │
│      // - Make HTTP request                                                 │
│      // - Parse response                                                    │
│      // - Set result                                                        │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Registration: Manually added to framework_provider.go.tmpl                 │
│                                                                             │
│  func (p *FrameworkProvider) EphemeralResources(_ context.Context)          │
│      []func() ephemeral.EphemeralResource {                                 │
│      return []func() ephemeral.EphemeralResource{                           │
│          // Each one manually listed                                        │
│          resourcemanager.GoogleEphemeralClientConfig,                       │
│          resourcemanager.GoogleEphemeralServiceAccountAccessToken,          │
│          secretmanager.GoogleEphemeralSecretManagerSecretVersion,           │
│          ...                                                                │
│      }                                                                      │
│  }                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Characteristics of Current Ephemeral Resources:**
- Uses **Plugin Framework** (`terraform-plugin-framework/ephemeral`)
- **Entirely handwritten** - no code generation
- **Duplicates API logic** that may exist in data source
- **Registered manually** in framework_provider.go.tmpl
- **State is NOT persisted** - data never written to state file
- **Sensitive fields manually marked** in schema

---

### Side-by-Side Comparison: Data Source vs Ephemeral (Current)

```
┌────────────────────────────────────┬────────────────────────────────────┐
│         DATA SOURCE (Current)       │      EPHEMERAL RESOURCE (Current)  │
├────────────────────────────────────┼────────────────────────────────────┤
│                                    │                                    │
│  ┌──────────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │     YAML Definition          │  │  │     NO YAML Definition       │  │
│  │  datasource_experimental:    │  │  │     (Handwritten only)       │  │
│  │    generate: true            │  │  │                              │  │
│  └──────────────────────────────┘  │  └──────────────────────────────┘  │
│               │                    │               │                    │
│               ▼                    │               ▼                    │
│  ┌──────────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │   mmv1 Generator             │  │  │   Manual Coding              │  │
│  │   (datasource.go.tmpl)       │  │  │   (in third_party/)          │  │
│  └──────────────────────────────┘  │  └──────────────────────────────┘  │
│               │                    │               │                    │
│               ▼                    │               ▼                    │
│  ┌──────────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │   SDK v2                     │  │  │   Plugin Framework           │  │
│  │   (terraform-plugin-sdk)     │  │  │   (terraform-plugin-framework)│  │
│  └──────────────────────────────┘  │  └──────────────────────────────┘  │
│               │                    │               │                    │
│               ▼                    │               ▼                    │
│  ┌──────────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │   Delegates to Resource      │  │  │   Standalone API Logic       │  │
│  │   resourceXxxRead(d, meta)   │  │  │   (duplicated code)          │  │
│  └──────────────────────────────┘  │  └──────────────────────────────┘  │
│               │                    │               │                    │
│               ▼                    │               ▼                    │
│  ┌──────────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │   Registry Registration      │  │  │   Manual Registration        │  │
│  │   registry.Schema{}.Register │  │  │   EphemeralResources() list  │  │
│  └──────────────────────────────┘  │  └──────────────────────────────┘  │
│               │                    │               │                    │
│               ▼                    │               ▼                    │
│  ┌──────────────────────────────┐  │  ┌──────────────────────────────┐  │
│  │   STATE FILE                 │  │  │   NO STATE                   │  │
│  │   (data persisted)           │  │  │   (never persisted)          │  │
│  └──────────────────────────────┘  │  └──────────────────────────────┘  │
│                                    │                                    │
└────────────────────────────────────┴────────────────────────────────────┘
```

---

### The Problem: Code Duplication

When both a data source and ephemeral resource exist for the same API (e.g., Secret Manager Secret Version), there's significant code duplication:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CODE DUPLICATION EXAMPLE                          │
│                     (Secret Manager Secret Version)                          │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────┐  ┌─────────────────────────────────────┐
│  data_source_secret_manager_        │  │  ephemeral_google_secret_manager_   │
│  secret_version.go (Handwritten)    │  │  secret_version.go (Handwritten)    │
├─────────────────────────────────────┤  ├─────────────────────────────────────┤
│                                     │  │                                     │
│  // Schema definition               │  │  // Schema definition               │
│  "project": {                       │  │  "project": schema.StringAttribute{ │
│      Type: schema.TypeString,       │  │      Optional: true,                │
│      Optional: true,                │  │      Computed: true,                │
│  },                                 │  │  },                                 │
│  "secret": {                        │  │  "secret": schema.StringAttribute{  │
│      Type: schema.TypeString,       │  │      Required: true,                │
│      Required: true,                │  │  },                                 │
│  },                                 │  │  "secret_data": schema.StringAttr{  │
│  "secret_data": {                   │  │      Computed: true,                │
│      Type: schema.TypeString,       │  │      Sensitive: true,               │
│      Computed: true,                │  │  },                                 │
│      Sensitive: true,               │  │                                     │
│  },                                 │  │  // DUPLICATED API LOGIC            │
│                                     │  │  url := fmt.Sprintf("%sprojects/    │
│  // API Logic                       │  │      %s/secrets/%s/versions/%s",    │
│  url := fmt.Sprintf("%sprojects/    │  │      config.SecretManagerBasePath,  │
│      %s/secrets/%s/versions/%s",    │  │      project, secret, version)      │
│      config.SecretManagerBasePath,  │  │                                     │
│      project, secret, version)      │  │  versionResp, err := transport_tpg. │
│                                     │  │      SendRequest(...)               │
│  versionResp, err := transport_tpg. │  │                                     │
│      SendRequest(...)               │  │  // DUPLICATED base64 decode        │
│                                     │  │  decoded, err := base64.StdEncoding.│
│  // base64 decode                   │  │      DecodeString(payload["data"])  │
│  decoded, err := base64.StdEncoding.│  │                                     │
│      DecodeString(payload["data"])  │  │                                     │
│                                     │  │                                     │
└─────────────────────────────────────┘  └─────────────────────────────────────┘
                     │                                      │
                     └──────────────┬───────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────────┐
                    │  ~70% of code is DUPLICATED:      │
                    │  - Schema field definitions       │
                    │  - URL construction               │
                    │  - API calls                      │
                    │  - Response parsing               │
                    │  - Base64 decoding                │
                    │  - Error handling                 │
                    │                                   │
                    │  ~30% is different:               │
                    │  - Framework types                │
                    │  - Registration mechanism         │
                    │  - State handling                 │
                    └───────────────────────────────────┘
```

---

### Why We Need a Unified Generator

| Problem | Impact |
|---------|--------|
| **Code duplication** | Same API logic written twice, bugs must be fixed twice |
| **Inconsistency risk** | Data source and ephemeral may behave differently |
| **Maintenance burden** | Changes require updating multiple files |
| **No YAML definition** | Ephemeral resources can't leverage existing resource definitions |
| **Manual registration** | Easy to forget to register new ephemeral resources |
| **Different frameworks** | SDK v2 vs Plugin Framework require different code |

**The Goal:** Generate BOTH from a single YAML definition, with shared core logic.

---

## Option A: Wrapper Pattern (Recommended)

### Concept

Generate ephemeral resources that wrap the underlying data source logic. The ephemeral resource calls a shared "core" function that performs the actual data fetching. This approach allows both SDK v2 (data sources) and Plugin Framework (ephemeral resources) to coexist while sharing the business logic.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           YAML Resource Definition                          │
│                    (mmv1/products/<product>/<Resource>.yaml)                │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            mmv1 Generator                                   │
│                         (mmv1/provider/terraform.go)                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   Core Logic File   │   │   Data Source File  │   │  Ephemeral File     │
│  (Framework Agnostic)│   │     (SDK v2)        │   │ (Plugin Framework)  │
│                     │   │                     │   │                     │
│ *_datasource_core.go│   │ data_source_*.go    │   │ ephemeral_*.go      │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
           │                         │                         │
           │                         ▼                         │
           │              ┌─────────────────────┐              │
           └─────────────▶│  API Calls / Data   │◀─────────────┘
                          │    Transformation   │
                          └─────────────────────┘
```

---

### Complete File Structure

For a resource named `SecretVersion` in product `secretmanager`:

```
mmv1/
├── products/
│   └── secretmanager/
│       └── SecretVersion.yaml                    # Resource definition (modified)
│
├── templates/
│   └── terraform/
│       ├── datasource.go.tmpl                    # Existing data source template
│       ├── datasource_core.go.tmpl               # NEW: Core logic template
│       ├── ephemeral.go.tmpl                     # NEW: Ephemeral resource template
│       ├── ephemeral_test.go.tmpl                # NEW: Ephemeral test template
│       └── examples/
│           └── base_configs/
│               └── ephemeral_test_file.go.tmpl   # NEW: Ephemeral test base
│
├── api/
│   └── resource/
│       ├── datasource.go                         # Modified: Add ephemeral fields
│       └── ephemeral.go                          # NEW: Ephemeral config struct
│
├── provider/
│   └── terraform.go                              # Modified: Add generation methods
│
└── third_party/
    └── terraform/
        └── fwprovider/
            ├── framework_provider.go.tmpl        # Modified: Register ephemeral
            └── generated_ephemeral_resources.go.tmpl  # NEW: Generated list

# Generated Output (in terraform-provider-google):
google/services/secretmanager/
├── secret_version_datasource_core.go             # NEW: Shared core logic
├── data_source_secret_version.go                 # Existing: SDK v2 data source
├── data_source_secret_version_test.go            # Existing: Data source tests
├── ephemeral_secret_version.go                   # NEW: Plugin Framework ephemeral
└── ephemeral_secret_version_test.go              # NEW: Ephemeral tests
```

---

### Critical Challenge: Type Conversion Between Frameworks

You're right to call this out - this is one of the most important technical challenges. SDK v2 and Plugin Framework use completely different type systems.

#### TL;DR - The Solution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         THE SOLUTION IN ONE DIAGRAM                          │
└─────────────────────────────────────────────────────────────────────────────┘

    SDK v2 Data Source                          Plugin Framework Ephemeral
    ==================                          ==========================

    schema.ResourceData                         model struct with types.String
           │                                              │
           │ d.Get("secret").(string)                     │ model.Secret.ValueString()
           │                                              │
           ▼                                              ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │                    CoreData (Plain Go Types)                            │
    │                                                                         │
    │    type SecretVersionCoreData struct {                                  │
    │        Project    string    // ← plain Go string, not SDK or PF type    │
    │        Secret     string                                                │
    │        SecretData string                                                │
    │        Enabled    bool      // ← plain Go bool                          │
    │    }                                                                    │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                                                                         │
    │              ReadSecretVersionCore(config, data) error                  │
    │                                                                         │
    │    // This function knows NOTHING about SDK v2 or Plugin Framework      │
    │    // It only works with plain Go types                                 │
    │    // Contains ALL the API logic (URL building, HTTP calls, parsing)    │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │                    CoreData (now populated)                             │
    └─────────────────────────────────────────────────────────────────────────┘
           │                                              │
           │ d.Set("secret_data", data.SecretData)        │ model.SecretData = types.StringValue(data.SecretData)
           │                                              │
           ▼                                              ▼
    schema.ResourceData                         model struct with types.String
    (stored in state)                           (NOT stored - ephemeral!)
```

**In short:**
1. **CoreData struct** uses plain Go types (`string`, `bool`, `int64`)
2. **Core function** does all API work using only plain Go types
3. **SDK v2 wrapper** converts `d.Get()` → CoreData → `d.Set()`
4. **Plugin Framework wrapper** converts `model.X.ValueString()` → CoreData → `types.StringValue()`

The frameworks never touch each other - they only talk to the plain Go CoreData struct.

---

#### Detailed Explanation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE TYPE CONVERSION PROBLEM                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│   SDK v2 Types                  │     │   Plugin Framework Types        │
│   (terraform-plugin-sdk)        │     │   (terraform-plugin-framework)  │
├─────────────────────────────────┤     ├─────────────────────────────────┤
│                                 │     │                                 │
│  schema.TypeString  → string    │     │  types.String  → has methods:   │
│  schema.TypeInt     → int       │     │    .ValueString()               │
│  schema.TypeBool    → bool      │     │    .IsNull()                    │
│  schema.TypeFloat   → float64   │     │    .IsUnknown()                 │
│  schema.TypeList    → []any     │     │                                 │
│  schema.TypeSet     → *Set      │     │  types.Int64   → has methods:   │
│  schema.TypeMap     → map       │     │    .ValueInt64()                │
│                                 │     │    .IsNull()                    │
│  Access via:                    │     │                                 │
│  d.Get("field").(string)        │     │  types.Bool    → has methods:   │
│  d.Set("field", value)          │     │    .ValueBool()                 │
│                                 │     │                                 │
│  No null/unknown distinction    │     │  types.List, types.Set, etc.    │
│  Empty string = not set         │     │                                 │
│                                 │     │  Access via model struct:       │
│                                 │     │  model.Field.ValueString()      │
└─────────────────────────────────┘     └─────────────────────────────────┘
                │                                       │
                │         INCOMPATIBLE!                 │
                └───────────────┬───────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │   How do we share     │
                    │   code between them?  │
                    └───────────────────────┘
```

---

### The Solution: Plain Go Types as Bridge (CoreData Struct)

The wrapper pattern solves this by using **plain Go types** (`string`, `bool`, `int64`, etc.) as an intermediate representation. Both frameworks convert to/from this common struct:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    THE COREDATA BRIDGE PATTERN                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│   SDK v2 Data Source            │     │   Plugin Framework Ephemeral    │
│   (schema.ResourceData)         │     │   (typed model struct)          │
└─────────────────────────────────┘     └─────────────────────────────────┘
                │                                       │
                │  CONVERT TO                           │  CONVERT TO
                ▼                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CoreData Struct (Plain Go Types - Framework Agnostic)                     │
│                                                                             │
│   type SecretVersionCoreData struct {                                       │
│       // Input fields - plain Go types                                      │
│       Project string                                                        │
│       Secret  string                                                        │
│       Version string                                                        │
│                                                                             │
│       // Output fields - plain Go types                                     │
│       Name        string                                                    │
│       SecretData  string                                                    │
│       CreateTime  string                                                    │
│       DestroyTime string                                                    │
│       Enabled     bool                                                      │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │  PASSED TO
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Core Function (Framework Agnostic)                                        │
│                                                                             │
│   func ReadSecretVersionCore(                                               │
│       config *transport_tpg.Config,                                         │
│       userAgent string,                                                     │
│       data *SecretVersionCoreData,    // ← Plain Go types                   │
│       opts *SecretVersionCoreConfig,                                        │
│   ) error {                                                                 │
│       // All API logic here - no framework dependencies                     │
│       // Works with plain strings, bools, etc.                              │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                │
                                │  RETURNS
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   CoreData Struct (now populated with results)                              │
└─────────────────────────────────────────────────────────────────────────────┘
                │                                       │
                │  CONVERT FROM                         │  CONVERT FROM
                ▼                                       ▼
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│   SDK v2 Data Source            │     │   Plugin Framework Ephemeral    │
│   d.Set("field", value)         │     │   model.Field = types.StringVal │
└─────────────────────────────────┘     └─────────────────────────────────┘
```

---

### Type Conversion Code Examples

**SDK v2 → CoreData (in data source):**

```go
func dataSourceSecretVersionRead(d *schema.ResourceData, meta interface{}) error {
    config := meta.(*transport_tpg.Config)

    // ┌─────────────────────────────────────────────────────────────────┐
    // │ CONVERT: SDK v2 schema.ResourceData → Plain Go types (CoreData) │
    // └─────────────────────────────────────────────────────────────────┘
    coreData := &SecretVersionCoreData{
        Project: d.Get("project").(string),      // SDK v2 type assertion
        Secret:  d.Get("secret").(string),       // SDK v2 type assertion
        Version: d.Get("version").(string),      // SDK v2 type assertion
    }

    // Call framework-agnostic core function
    if err := ReadSecretVersionCore(config, userAgent, coreData, nil); err != nil {
        return err
    }

    // ┌─────────────────────────────────────────────────────────────────┐
    // │ CONVERT: Plain Go types (CoreData) → SDK v2 schema.ResourceData │
    // └─────────────────────────────────────────────────────────────────┘
    d.SetId(coreData.Name)
    d.Set("name", coreData.Name)                 // string → SDK v2
    d.Set("secret_data", coreData.SecretData)    // string → SDK v2
    d.Set("enabled", coreData.Enabled)           // bool → SDK v2

    return nil
}
```

**Plugin Framework → CoreData (in ephemeral resource):**

```go
func (e *ephemeralSecretVersion) Open(ctx context.Context, req ephemeral.OpenRequest, resp *ephemeral.OpenResponse) {
    var model ephemeralSecretVersionModel
    resp.Diagnostics.Append(req.Config.Get(ctx, &model)...)

    // ┌─────────────────────────────────────────────────────────────────┐
    // │ CONVERT: Plugin Framework types.String → Plain Go string        │
    // └─────────────────────────────────────────────────────────────────┘
    coreData := &SecretVersionCoreData{
        Project: model.Project.ValueString(),    // types.String → string
        Secret:  model.Secret.ValueString(),     // types.String → string
        Version: model.Version.ValueString(),    // types.String → string
    }

    // Call framework-agnostic core function
    if err := ReadSecretVersionCore(config, userAgent, coreData, nil); err != nil {
        resp.Diagnostics.AddError("Error", err.Error())
        return
    }

    // ┌─────────────────────────────────────────────────────────────────┐
    // │ CONVERT: Plain Go string → Plugin Framework types.String        │
    // └─────────────────────────────────────────────────────────────────┘
    model.Name = types.StringValue(coreData.Name)           // string → types.String
    model.SecretData = types.StringValue(coreData.SecretData)
    model.Enabled = types.BoolValue(coreData.Enabled)       // bool → types.Bool

    // Handle null vs empty (Plugin Framework distinction)
    if coreData.DestroyTime != "" {
        model.DestroyTime = types.StringValue(coreData.DestroyTime)
    } else {
        model.DestroyTime = types.StringNull()  // Explicit null
    }

    resp.Diagnostics.Append(resp.Result.Set(ctx, model)...)
}
```

---

### Handling Complex Types

The conversion is straightforward for primitives, but what about complex types like lists, maps, and nested objects?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLEX TYPE CONVERSION STRATEGIES                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────┬─────────────────────────────────┬─────────┐
│   Type                          │   CoreData Representation       │  Notes  │
├─────────────────────────────────┼─────────────────────────────────┼─────────┤
│   String                        │   string                        │  Simple │
│   Integer                       │   int64                         │  Simple │
│   Boolean                       │   bool                          │  Simple │
│   Float                         │   float64                       │  Simple │
├─────────────────────────────────┼─────────────────────────────────┼─────────┤
│   List of Strings               │   []string                      │  Easy   │
│   Set of Strings                │   []string (order ignored)      │  Easy   │
│   Map of Strings                │   map[string]string             │  Easy   │
├─────────────────────────────────┼─────────────────────────────────┼─────────┤
│   List of Objects               │   []NestedCoreData              │  Medium │
│   Nested Object                 │   *NestedCoreData               │  Medium │
├─────────────────────────────────┼─────────────────────────────────┼─────────┤
│   Deeply Nested / Complex       │   map[string]interface{}        │  Hard   │
│                                 │   (use JSON marshaling)         │         │
└─────────────────────────────────┴─────────────────────────────────┴─────────┘
```

**Example: List of Strings**

```go
// CoreData struct
type SecretVersionCoreData struct {
    Labels []string  // Plain Go slice
}

// SDK v2 → CoreData
labels := d.Get("labels").([]interface{})
coreData.Labels = make([]string, len(labels))
for i, v := range labels {
    coreData.Labels[i] = v.(string)
}

// Plugin Framework → CoreData
var labelsList []string
model.Labels.ElementsAs(ctx, &labelsList, false)
coreData.Labels = labelsList

// CoreData → SDK v2
d.Set("labels", coreData.Labels)

// CoreData → Plugin Framework
labelValues := make([]attr.Value, len(coreData.Labels))
for i, v := range coreData.Labels {
    labelValues[i] = types.StringValue(v)
}
model.Labels, _ = types.ListValue(types.StringType, labelValues)
```

**Example: Nested Object**

```go
// CoreData structs
type SecretVersionCoreData struct {
    Replication *ReplicationCoreData
}

type ReplicationCoreData struct {
    Automatic   bool
    UserManaged *UserManagedCoreData
}

// The conversion follows the same pattern recursively
```

---

### Why Plain Go Types Work

| Consideration | Plain Go Types Solution |
|--------------|------------------------|
| **Null handling** | Use pointers (`*string`) or empty values + separate flag |
| **Unknown handling** | Not applicable at read time (values are known) |
| **Type safety** | Go compiler enforces types |
| **Serialization** | Easy to JSON marshal for debugging |
| **Testing** | Can unit test core function with plain Go values |
| **Performance** | Minimal overhead - just value copying |

---

### Handling Null vs Empty (Plugin Framework Distinction)

Plugin Framework distinguishes between null, unknown, and empty values. SDK v2 does not. Here's how to handle this:

```go
// In CoreData, use pointers for nullable fields
type SecretVersionCoreData struct {
    DestroyTime *string  // nil = null, "" = empty string, "value" = has value
}

// Or use a separate "is set" flag
type SecretVersionCoreData struct {
    DestroyTime      string
    DestroyTimeIsSet bool
}

// Plugin Framework conversion
if coreData.DestroyTime != nil {
    model.DestroyTime = types.StringValue(*coreData.DestroyTime)
} else {
    model.DestroyTime = types.StringNull()
}

// SDK v2 conversion (doesn't distinguish null vs empty)
if coreData.DestroyTime != nil {
    d.Set("destroy_time", *coreData.DestroyTime)
}
```

---

### Potential Problems and Open Questions

This approach has trade-offs and challenges that need to be addressed:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POTENTIAL PROBLEMS WITH THIS APPROACH                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 1. Null vs Unknown vs Empty - The Hardest Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   Plugin Framework has THREE states:                                        │
│                                                                             │
│   types.StringNull()     → Field was not set (null in JSON)                 │
│   types.StringUnknown()  → Field value not yet known (during plan)          │
│   types.StringValue("")  → Field was explicitly set to empty string         │
│                                                                             │
│   SDK v2 only has TWO:                                                      │
│                                                                             │
│   "" (empty string)      → Could mean null OR empty, ambiguous!             │
│   "value"                → Has a value                                      │
│                                                                             │
│   Plain Go has TWO (without pointers):                                      │
│                                                                             │
│   "" (zero value)        → Could mean null OR empty                         │
│   "value"                → Has a value                                      │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: How do we preserve null vs empty distinction through CoreData?

SOLUTION OPTIONS:
  A) Use pointers: *string (nil = null, "" = empty, "val" = value)
  B) Use wrapper: type NullableString struct { Value string; IsNull bool }
  C) Ignore it: Most fields don't need this distinction for read-only ops
  D) Per-field decision: Only use pointers for fields where it matters
```

#### 2. Unknown Values During Planning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Plugin Framework can have "unknown" values during terraform plan │
│                                                                             │
│   ephemeral "google_secret" "x" {                                           │
│       secret = google_secret.new_secret.name  # Unknown until apply!        │
│   }                                                                         │
│                                                                             │
│   During plan:                                                              │
│     model.Secret.IsUnknown() == true                                        │
│     model.Secret.ValueString() == ""  # Returns empty, not the real value! │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: What happens if we call the core function with unknown values?

ANSWER: For ephemeral resources, Open() is only called during apply, not plan.
        So unknown values should be resolved by the time Open() runs.
        But we should still check and handle gracefully:

        if model.Secret.IsUnknown() {
            resp.Diagnostics.AddError("Cannot read", "Secret is not yet known")
            return
        }
```

#### 3. Complex Nested Types Get Messy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Deeply nested structures require lots of conversion code         │
│                                                                             │
│   Example: A resource with nested blocks                                    │
│                                                                             │
│   properties:                                                               │
│     - name: 'replication'                                                   │
│       type: NestedObject                                                    │
│       properties:                                                           │
│         - name: 'replicas'                                                  │
│           type: Array                                                       │
│           item_type:                                                        │
│             type: NestedObject                                              │
│             properties:                                                     │
│               - name: 'location'                                            │
│                 type: String                                                │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: Do we generate conversion code for every nested level?

SOLUTION OPTIONS:
  A) Generate recursive CoreData structs and conversion functions
  B) Use map[string]interface{} for complex nested data (lose type safety)
  C) Limit ephemeral generation to "simple" resources only
  D) Use JSON marshal/unmarshal as intermediate (performance cost)
```

#### 4. Validation Logic Duplication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Validators are framework-specific                                │
│                                                                             │
│   SDK v2:                                                                   │
│     ValidateFunc: validation.StringInSlice([]string{"A", "B"}, false)       │
│                                                                             │
│   Plugin Framework:                                                         │
│     Validators: []validator.String{stringvalidator.OneOf("A", "B")}         │
│                                                                             │
│   These can't be shared!                                                    │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: Do we duplicate validation in both places?

SOLUTION OPTIONS:
  A) Generate framework-specific validators from YAML validation rules
  B) Do validation in core function (after type conversion) - adds latency
  C) Accept duplication - validation code is usually small
  D) Only validate in core, skip framework validators
```

#### 5. Error Handling Differences

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Different error patterns                                         │
│                                                                             │
│   SDK v2:          return fmt.Errorf("failed: %s", err)                     │
│   Plugin Framework: resp.Diagnostics.AddError("Title", "Detail")            │
│   Core function:    return error (plain Go)                                 │
│                                                                             │
│   Core returns error, wrappers must convert to their format                 │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: How do we provide good error messages from core?

SOLUTION: Core returns structured errors that wrappers can format:

  type CoreError struct {
      Operation string  // "reading secret version"
      Detail    string  // "API returned 404"
      Err       error   // underlying error
  }

  // SDK v2 wrapper
  if err != nil {
      return fmt.Errorf("Error %s: %s", err.Operation, err.Detail)
  }

  // Plugin Framework wrapper
  if err != nil {
      resp.Diagnostics.AddError(
          fmt.Sprintf("Error %s", err.Operation),
          err.Detail,
      )
  }
```

#### 6. Custom Code Injection

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Some resources need custom logic that can't be generated         │
│                                                                             │
│   Example: Special API call patterns, custom encoding, workarounds          │
│                                                                             │
│   Current resources have:                                                   │
│     custom_code:                                                            │
│       encoder: 'templates/terraform/encoders/my_resource.go.tmpl'           │
│       pre_read: 'templates/terraform/pre_read/my_resource.go.tmpl'          │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: How do we allow custom code in the core function?

SOLUTION OPTIONS:
  A) Custom code templates that inject into core function
  B) Pre/post hooks: PreReadCore(), PostReadCore()
  C) Override entire core function with custom template
  D) Limit ephemeral generation to resources without custom code
```

#### 7. Performance Overhead

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Extra conversion steps add overhead                              │
│                                                                             │
│   Before (direct):                                                          │
│     API Response → SDK v2 types                                             │
│                                                                             │
│   After (with core):                                                        │
│     API Response → CoreData → SDK v2 types                                  │
│     API Response → CoreData → Plugin Framework types                        │
│                                                                             │
│   Extra allocations and copies                                              │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: Is the performance hit acceptable?

ANSWER: Probably yes, because:
  - Read operations are I/O bound (API calls), not CPU bound
  - CoreData structs are small
  - Only adds microseconds, API calls take milliseconds
  - Trade-off is worth it for code maintainability
```

#### 8. Testing Complexity

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: More moving parts = more things to test                          │
│                                                                             │
│   Need to test:                                                             │
│     1. Core function (unit tests with plain Go)                             │
│     2. SDK v2 → CoreData conversion                                         │
│     3. CoreData → SDK v2 conversion                                         │
│     4. Plugin Framework → CoreData conversion                               │
│     5. CoreData → Plugin Framework conversion                               │
│     6. End-to-end acceptance tests                                          │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: How do we test all these layers?

SOLUTION:
  - Core function: Unit tests with mock HTTP responses
  - Conversion: Generate test helpers alongside conversion code
  - E2E: Existing acceptance test patterns still work
  - Benefit: Core function is easier to unit test than framework-specific code
```

#### 9. What About Data Sources That Already Exist?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│   PROBLEM: Many data sources are handwritten, not generated                 │
│                                                                             │
│   Examples:                                                                 │
│     data_source_secret_manager_secret_version.go  (handwritten)             │
│     data_source_google_client_config.go           (handwritten)             │
│     data_source_compute_instance.go               (handwritten)             │
│                                                                             │
│   These don't use the datasource.go.tmpl template                           │
└─────────────────────────────────────────────────────────────────────────────┘

QUESTION: How do we add ephemeral resources for handwritten data sources?

SOLUTION OPTIONS:
  A) Refactor handwritten data sources to use core pattern (breaking change)
  B) Write ephemeral resources by hand (defeats the purpose)
  C) Extract core from existing handwritten code into separate file
  D) Only support ephemeral generation for YAML-defined resources
```

---

### Summary: Is This Approach Worth It?

| Challenge | Severity | Mitigation |
|-----------|----------|------------|
| Null vs Empty distinction | Medium | Use pointers for nullable fields |
| Unknown values | Low | Only affects plan-time, ephemeral is apply-time |
| Nested types | High | Start with simple resources, add complexity later |
| Validation duplication | Low | Generate from YAML, small code anyway |
| Error handling | Low | Use structured errors in core |
| Custom code | Medium | Support injection points in templates |
| Performance | Low | Negligible compared to API latency |
| Testing | Medium | Core is actually easier to unit test |
| Handwritten data sources | High | Case-by-case migration |

**Verdict:** The approach is sound, but start with simple resources and iterate:

1. **Phase 1:** Generate for simple resources (few fields, no nesting)
2. **Phase 2:** Add support for lists and maps
3. **Phase 3:** Add support for nested objects
4. **Phase 4:** Migration path for handwritten data sources

---

### Generation Flow (Step by Step)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: Load YAML Definition                                                │
│ File: mmv1/loader/loader.go                                                 │
│                                                                             │
│ - Reads SecretVersion.yaml                                                  │
│ - Parses datasource_experimental block                                      │
│ - Parses ephemeral_experimental block (NEW)                                 │
│ - Creates api.Resource struct with all configuration                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: Check Generation Flags                                              │
│ File: mmv1/api/resource.go                                                  │
│                                                                             │
│ ShouldGenerateSingularDataSource()    → true (existing)                     │
│ ShouldGenerateDataSourceCore()        → true (NEW - if ephemeral enabled)   │
│ ShouldGenerateEphemeralResource()     → true (NEW)                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: Generate Files                                                      │
│ File: mmv1/provider/terraform.go                                            │
│                                                                             │
│ GenerateObject() calls:                                                     │
│   ├── GenerateDataSourceCore()        → secret_version_datasource_core.go   │
│   ├── GenerateSingularDataSource()    → data_source_secret_version.go       │
│   ├── GenerateEphemeralResource()     → ephemeral_secret_version.go         │
│   └── GenerateEphemeralResourceTests()→ ephemeral_secret_version_test.go    │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: Generate Provider Registration                                      │
│ File: mmv1/provider/terraform.go (CompileCommonFiles)                       │
│                                                                             │
│ - Collects all resources with ephemeral_experimental.generate = true        │
│ - Generates generated_ephemeral_resources.go with list of constructors      │
│ - framework_provider.go.tmpl includes this list in EphemeralResources()     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### YAML Configuration (Detailed)

**File:** `mmv1/products/secretmanager/SecretVersion.yaml`

```yaml
# Copyright 2024 Google Inc.
# Licensed under the Apache License, Version 2.0 (the "License");
name: 'SecretVersion'
description: |
  A Secret Version resource. Secret Manager versions hold the actual secret data.
  This can be used as a data source to read secret values, or as an ephemeral
  resource to access secrets without persisting them to state.

references:
  guides:
    'Managing Secrets': 'https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets'
  api: 'https://cloud.google.com/secret-manager/docs/reference/rest/v1/projects.secrets.versions'

base_url: 'projects/{{project}}/secrets/{{secret}}/versions'
self_link: 'projects/{{project}}/secrets/{{secret}}/versions/{{version}}'
id_format: 'projects/{{project}}/secrets/{{secret}}/versions/{{version}}'

# ============================================================================
# DATA SOURCE CONFIGURATION (Existing)
# ============================================================================
datasource_experimental:
  # Generate the SDK v2 data source
  generate: true

  # Generate tests for the data source
  exclude_test: false

# ============================================================================
# EPHEMERAL RESOURCE CONFIGURATION (NEW)
# ============================================================================
ephemeral_experimental:
  # Generate the Plugin Framework ephemeral resource
  generate: true

  # Generate tests for the ephemeral resource
  exclude_test: false

  # Fields that should be marked as Sensitive in the ephemeral resource
  # These fields will have `Sensitive: true` in the Plugin Framework schema
  sensitive_fields:
    - 'secretData'

  # Optional: Fields that should be excluded from the ephemeral resource
  # (e.g., fields only relevant for state management)
  exclude_fields: []

  # Optional: Additional description for the ephemeral resource
  # If not provided, uses the resource description
  description: |
    Provides ephemeral access to a secret version's data. The secret data
    is never stored in Terraform state or plan files.

  # Optional: Custom example for ephemeral resource documentation
  example: |
    ephemeral "google_secret_manager_secret_version" "my_secret" {
      secret  = "my-secret"
      version = "latest"
    }

    # Use in provider configuration
    provider "example" {
      api_key = ephemeral.google_secret_manager_secret_version.my_secret.secret_data
    }

# ============================================================================
# PROPERTIES
# ============================================================================
properties:
  # Input: Project ID
  - name: 'project'
    type: String
    description: |
      The project containing the secret. If not provided, the provider project is used.
    url_param_only: true
    default_from_api: true

  # Input: Secret name/ID
  - name: 'secret'
    type: String
    description: |
      The secret to retrieve the version for. Can be the secret ID or the full
      resource name (projects/PROJECT/secrets/SECRET).
    required: true
    url_param_only: true

  # Input: Version (optional, defaults to "latest")
  - name: 'version'
    type: String
    description: |
      The version of the secret to retrieve. If not specified, the latest
      version is retrieved.
    url_param_only: true
    default_value: 'latest'

  # Output: Full resource name
  - name: 'name'
    type: String
    description: |
      The resource name of the SecretVersion.
      Format: projects/{{project}}/secrets/{{secret_id}}/versions/{{version}}
    output: true

  # Output: Secret data (SENSITIVE)
  - name: 'secretData'
    type: String
    description: |
      The secret data. No larger than 64KiB.
    output: true
    # Note: This is automatically marked sensitive in ephemeral via sensitive_fields

  # Output: Creation time
  - name: 'createTime'
    type: String
    description: |
      The time at which the Secret was created.
    output: true

  # Output: Destruction time (if destroyed)
  - name: 'destroyTime'
    type: String
    description: |
      The time at which the Secret was destroyed. Only present if state is DESTROYED.
    output: true

  # Output: Enabled state
  - name: 'enabled'
    type: Boolean
    description: |
      True if the current state of the SecretVersion is enabled.
    output: true
```

---

### Go Struct Definitions (New/Modified Files)

**File:** `mmv1/api/resource/ephemeral.go` (NEW)

```go
// Copyright 2024 Google Inc.
// Licensed under the Apache License, Version 2.0 (the "License");

package resource

// Ephemeral contains configuration for generating ephemeral resources
type Ephemeral struct {
    // Generate determines whether to generate the ephemeral resource
    Generate bool `yaml:"generate"`

    // ExcludeTest determines whether to skip test generation
    ExcludeTest bool `yaml:"exclude_test"`

    // SensitiveFields lists field names that should be marked as Sensitive
    // in the Plugin Framework schema
    SensitiveFields []string `yaml:"sensitive_fields"`

    // ExcludeFields lists field names that should not be included in the
    // ephemeral resource (e.g., state-only fields)
    ExcludeFields []string `yaml:"exclude_fields"`

    // Description overrides the resource description for the ephemeral resource
    Description string `yaml:"description,omitempty"`

    // Example provides a custom example for documentation
    Example string `yaml:"example,omitempty"`
}

// IsSensitiveField checks if a field should be marked as sensitive
func (e *Ephemeral) IsSensitiveField(fieldName string) bool {
    for _, f := range e.SensitiveFields {
        if f == fieldName {
            return true
        }
    }
    return false
}

// IsExcludedField checks if a field should be excluded from the ephemeral resource
func (e *Ephemeral) IsExcludedField(fieldName string) bool {
    for _, f := range e.ExcludeFields {
        if f == fieldName {
            return true
        }
    }
    return false
}
```

**File:** `mmv1/api/resource.go` (MODIFICATIONS)

```go
// Add to Resource struct (around line 36)
type Resource struct {
    // ... existing fields ...

    // Datasource contains configuration for generating singular data sources
    Datasource *Datasource `yaml:"datasource_experimental,omitempty"`

    // Ephemeral contains configuration for generating ephemeral resources (NEW)
    Ephemeral *Ephemeral `yaml:"ephemeral_experimental,omitempty"`

    // ... rest of existing fields ...
}

// Add new methods (after ShouldGenerateSingularDataSource around line 2448)

// ShouldGenerateEphemeralResource returns true if an ephemeral resource should be generated
func (r *Resource) ShouldGenerateEphemeralResource() bool {
    if r.Ephemeral == nil {
        return false
    }
    return r.Ephemeral.Generate
}

// ShouldGenerateEphemeralResourceTests returns true if ephemeral tests should be generated
func (r *Resource) ShouldGenerateEphemeralResourceTests() bool {
    if r.Ephemeral == nil {
        return false
    }
    return r.Ephemeral.Generate && !r.Ephemeral.ExcludeTest
}

// ShouldGenerateDataSourceCore returns true if a shared core file should be generated
// This is needed when both data source and ephemeral resource are generated
func (r *Resource) ShouldGenerateDataSourceCore() bool {
    return r.ShouldGenerateSingularDataSource() && r.ShouldGenerateEphemeralResource()
}

// EphemeralDescription returns the description for the ephemeral resource
func (r *Resource) EphemeralDescription() string {
    if r.Ephemeral != nil && r.Ephemeral.Description != "" {
        return r.Ephemeral.Description
    }
    return r.Description
}

// IsEphemeralSensitiveField checks if a property should be sensitive in ephemeral resource
func (r *Resource) IsEphemeralSensitiveField(propertyName string) bool {
    if r.Ephemeral == nil {
        return false
    }
    return r.Ephemeral.IsSensitiveField(propertyName)
}

// EphemeralProperties returns properties filtered for the ephemeral resource
func (r *Resource) EphemeralProperties() []*Type {
    if r.Ephemeral == nil {
        return r.Properties
    }

    var props []*Type
    for _, p := range r.Properties {
        if !r.Ephemeral.IsExcludedField(p.Name) {
            props = append(props, p)
        }
    }
    return props
}
```

---

### Generated Code: Core Logic File

**Template:** `mmv1/templates/terraform/datasource_core.go.tmpl` (NEW)

**Generated Output:** `google/services/secretmanager/secret_version_datasource_core.go`

```go
// Copyright 2024 Google Inc.
// Code generated by Magic Modules. DO NOT EDIT.

package secretmanager

import (
    "encoding/base64"
    "fmt"
    "regexp"

    "github.com/hashicorp/terraform-provider-google/google/tpgresource"
    transport_tpg "github.com/hashicorp/terraform-provider-google/google/transport"
)

// SecretVersionCoreData holds the data retrieved from the API
// This is a framework-agnostic struct that both SDK v2 and Plugin Framework can use
type SecretVersionCoreData struct {
    // Input fields
    Project string
    Secret  string
    Version string

    // Output fields
    Name        string
    SecretData  string
    CreateTime  string
    DestroyTime string
    Enabled     bool

    // Computed from response
    ResolvedProject string
    ResolvedVersion string
}

// SecretVersionCoreConfig holds configuration for the core read operation
type SecretVersionCoreConfig struct {
    // IsSecretDataBase64 determines if secret data should be returned as base64
    IsSecretDataBase64 bool
}

// ReadSecretVersionCore performs the actual API calls to read a secret version
// This function is framework-agnostic and can be called by both data source and ephemeral resource
func ReadSecretVersionCore(
    config *transport_tpg.Config,
    userAgent string,
    data *SecretVersionCoreData,
    opts *SecretVersionCoreConfig,
) error {
    if opts == nil {
        opts = &SecretVersionCoreConfig{}
    }

    project := data.Project
    if project == "" {
        project = config.Project
    }
    data.ResolvedProject = project

    secret := data.Secret
    version := data.Version
    if version == "" {
        version = "latest"
    }

    // Build the URL for getting version metadata
    url := fmt.Sprintf("%sprojects/%s/secrets/%s/versions/%s",
        config.SecretManagerBasePath, project, secret, version)

    // Get version metadata
    versionResp, err := transport_tpg.SendRequest(transport_tpg.SendRequestOptions{
        Config:    config,
        Method:    "GET",
        Project:   project,
        RawURL:    url,
        UserAgent: userAgent,
    })
    if err != nil {
        return fmt.Errorf("error retrieving secret version: %s", err)
    }

    // Parse the response
    nameValue, ok := versionResp["name"].(string)
    if !ok {
        return fmt.Errorf("response missing 'name' field")
    }

    // Extract version number from name
    secretVersionRegex := regexp.MustCompile(`projects/(.+)/secrets/(.+)/versions/(.+)$`)
    parts := secretVersionRegex.FindStringSubmatch(nameValue)
    if len(parts) != 4 {
        return fmt.Errorf("invalid secret version name format: %s", nameValue)
    }
    data.ResolvedVersion = parts[3]

    // Set metadata fields
    data.Name = nameValue
    if createTime, ok := versionResp["createTime"].(string); ok {
        data.CreateTime = createTime
    }
    if destroyTime, ok := versionResp["destroyTime"].(string); ok {
        data.DestroyTime = destroyTime
    }
    data.Enabled = true // If we can read it, it's enabled

    // Access the secret data
    accessURL := fmt.Sprintf("%s:access", url)
    accessResp, err := transport_tpg.SendRequest(transport_tpg.SendRequestOptions{
        Config:    config,
        Method:    "GET",
        Project:   project,
        RawURL:    accessURL,
        UserAgent: userAgent,
    })
    if err != nil {
        return fmt.Errorf("error accessing secret data: %s", err)
    }

    // Extract and decode payload
    payload, ok := accessResp["payload"].(map[string]interface{})
    if !ok {
        return fmt.Errorf("response missing 'payload' field")
    }

    payloadData, ok := payload["data"].(string)
    if !ok {
        return fmt.Errorf("payload missing 'data' field")
    }

    // Decode if not requesting base64
    if !opts.IsSecretDataBase64 {
        decoded, err := base64.StdEncoding.DecodeString(payloadData)
        if err != nil {
            return fmt.Errorf("error decoding secret data: %s", err)
        }
        payloadData = string(decoded)
    }
    data.SecretData = payloadData

    return nil
}
```

---

### Generated Code: Data Source (Modified)

**Template:** `mmv1/templates/terraform/datasource.go.tmpl` (MODIFIED)

**Generated Output:** `google/services/secretmanager/data_source_secret_version.go`

```go
// Copyright 2024 Google Inc.
// Code generated by Magic Modules. DO NOT EDIT.

package secretmanager

import (
    "fmt"

    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"

    "github.com/hashicorp/terraform-provider-google/google/registry"
    "github.com/hashicorp/terraform-provider-google/google/tpgresource"
    transport_tpg "github.com/hashicorp/terraform-provider-google/google/transport"
)

func init() {
    registry.Schema{
        Name:        "google_secret_manager_secret_version",
        ProductName: "secretmanager",
        Type:        registry.SchemaTypeDataSource,
        Schema:      DataSourceSecretManagerSecretVersion(),
    }.Register()
}

func DataSourceSecretManagerSecretVersion() *schema.Resource {
    return &schema.Resource{
        Read: dataSourceSecretManagerSecretVersionRead,
        Schema: map[string]*schema.Schema{
            "project": {
                Type:        schema.TypeString,
                Optional:    true,
                Computed:    true,
                Description: "The project containing the secret.",
            },
            "secret": {
                Type:        schema.TypeString,
                Required:    true,
                Description: "The secret to retrieve the version for.",
            },
            "version": {
                Type:        schema.TypeString,
                Optional:    true,
                Computed:    true,
                Description: "The version of the secret to retrieve.",
            },
            "is_secret_data_base64": {
                Type:        schema.TypeBool,
                Optional:    true,
                Default:     false,
                Description: "If true, the secret data is returned as base64.",
            },
            "name": {
                Type:        schema.TypeString,
                Computed:    true,
                Description: "The resource name of the SecretVersion.",
            },
            "secret_data": {
                Type:        schema.TypeString,
                Computed:    true,
                Sensitive:   true,
                Description: "The secret data.",
            },
            "create_time": {
                Type:        schema.TypeString,
                Computed:    true,
                Description: "The time at which the Secret was created.",
            },
            "destroy_time": {
                Type:        schema.TypeString,
                Computed:    true,
                Description: "The time at which the Secret was destroyed.",
            },
            "enabled": {
                Type:        schema.TypeBool,
                Computed:    true,
                Description: "True if the current state is enabled.",
            },
        },
    }
}

func dataSourceSecretManagerSecretVersionRead(d *schema.ResourceData, meta interface{}) error {
    config := meta.(*transport_tpg.Config)
    userAgent, err := tpgresource.GenerateUserAgentString(d, config.UserAgent)
    if err != nil {
        return err
    }

    // Prepare core data from schema.ResourceData
    coreData := &SecretVersionCoreData{
        Project: d.Get("project").(string),
        Secret:  d.Get("secret").(string),
        Version: d.Get("version").(string),
    }

    coreOpts := &SecretVersionCoreConfig{
        IsSecretDataBase64: d.Get("is_secret_data_base64").(bool),
    }

    // Call the shared core function
    if err := ReadSecretVersionCore(config, userAgent, coreData, coreOpts); err != nil {
        return err
    }

    // Set the ID
    d.SetId(coreData.Name)

    // Map core data back to schema.ResourceData
    if err := d.Set("project", coreData.ResolvedProject); err != nil {
        return fmt.Errorf("error setting project: %s", err)
    }
    if err := d.Set("version", coreData.ResolvedVersion); err != nil {
        return fmt.Errorf("error setting version: %s", err)
    }
    if err := d.Set("name", coreData.Name); err != nil {
        return fmt.Errorf("error setting name: %s", err)
    }
    if err := d.Set("secret_data", coreData.SecretData); err != nil {
        return fmt.Errorf("error setting secret_data: %s", err)
    }
    if err := d.Set("create_time", coreData.CreateTime); err != nil {
        return fmt.Errorf("error setting create_time: %s", err)
    }
    if err := d.Set("destroy_time", coreData.DestroyTime); err != nil {
        return fmt.Errorf("error setting destroy_time: %s", err)
    }
    if err := d.Set("enabled", coreData.Enabled); err != nil {
        return fmt.Errorf("error setting enabled: %s", err)
    }

    return nil
}
```

---

### Generated Code: Ephemeral Resource

**Template:** `mmv1/templates/terraform/ephemeral.go.tmpl` (NEW)

**Generated Output:** `google/services/secretmanager/ephemeral_secret_version.go`

```go
// Copyright 2024 Google Inc.
// Code generated by Magic Modules. DO NOT EDIT.

package secretmanager

import (
    "context"
    "fmt"

    "github.com/hashicorp/terraform-plugin-framework/ephemeral"
    "github.com/hashicorp/terraform-plugin-framework/ephemeral/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"

    "github.com/hashicorp/terraform-provider-google/google/fwresource"
    transport_tpg "github.com/hashicorp/terraform-provider-google/google/transport"
)

// Ensure interface compliance
var _ ephemeral.EphemeralResource = &ephemeralSecretManagerSecretVersion{}
var _ ephemeral.EphemeralResourceWithConfigure = &ephemeralSecretManagerSecretVersion{}

// EphemeralSecretManagerSecretVersion returns the ephemeral resource constructor
func EphemeralSecretManagerSecretVersion() ephemeral.EphemeralResource {
    return &ephemeralSecretManagerSecretVersion{}
}

// ephemeralSecretManagerSecretVersion implements the ephemeral resource
type ephemeralSecretManagerSecretVersion struct {
    providerConfig *transport_tpg.Config
}

// ephemeralSecretManagerSecretVersionModel is the Plugin Framework model
type ephemeralSecretManagerSecretVersionModel struct {
    // Input attributes
    Project            types.String `tfsdk:"project"`
    Secret             types.String `tfsdk:"secret"`
    Version            types.String `tfsdk:"version"`
    IsSecretDataBase64 types.Bool   `tfsdk:"is_secret_data_base64"`

    // Output attributes
    Name        types.String `tfsdk:"name"`
    SecretData  types.String `tfsdk:"secret_data"`
    CreateTime  types.String `tfsdk:"create_time"`
    DestroyTime types.String `tfsdk:"destroy_time"`
    Enabled     types.Bool   `tfsdk:"enabled"`
}

// Metadata returns the resource type name
func (e *ephemeralSecretManagerSecretVersion) Metadata(
    ctx context.Context,
    req ephemeral.MetadataRequest,
    resp *ephemeral.MetadataResponse,
) {
    resp.TypeName = req.ProviderTypeName + "_secret_manager_secret_version"
}

// Schema returns the Plugin Framework schema
func (e *ephemeralSecretManagerSecretVersion) Schema(
    ctx context.Context,
    req ephemeral.SchemaRequest,
    resp *ephemeral.SchemaResponse,
) {
    resp.Schema = schema.Schema{
        Description: "Provides ephemeral access to a secret version's data. " +
            "The secret data is never stored in Terraform state or plan files.",
        MarkdownDescription: "Provides ephemeral access to a secret version's data. " +
            "The secret data is **never stored** in Terraform state or plan files.",
        Attributes: map[string]schema.Attribute{
            // Input attributes
            "project": schema.StringAttribute{
                Description: "The project containing the secret. If not provided, the provider project is used.",
                Optional:    true,
                Computed:    true,
            },
            "secret": schema.StringAttribute{
                Description: "The secret to retrieve the version for.",
                Required:    true,
            },
            "version": schema.StringAttribute{
                Description: "The version of the secret to retrieve. Defaults to 'latest'.",
                Optional:    true,
                Computed:    true,
            },
            "is_secret_data_base64": schema.BoolAttribute{
                Description: "If true, the secret data is returned as base64 without decoding.",
                Optional:    true,
            },

            // Output attributes
            "name": schema.StringAttribute{
                Description: "The resource name of the SecretVersion.",
                Computed:    true,
            },
            "secret_data": schema.StringAttribute{
                Description: "The secret data. No larger than 64KiB.",
                Computed:    true,
                Sensitive:   true, // <-- Marked sensitive for ephemeral
            },
            "create_time": schema.StringAttribute{
                Description: "The time at which the Secret was created.",
                Computed:    true,
            },
            "destroy_time": schema.StringAttribute{
                Description: "The time at which the Secret was destroyed.",
                Computed:    true,
            },
            "enabled": schema.BoolAttribute{
                Description: "True if the current state is enabled.",
                Computed:    true,
            },
        },
    }
}

// Configure receives the provider configuration
func (e *ephemeralSecretManagerSecretVersion) Configure(
    ctx context.Context,
    req ephemeral.ConfigureRequest,
    resp *ephemeral.ConfigureResponse,
) {
    if req.ProviderData == nil {
        return
    }

    config, ok := req.ProviderData.(*transport_tpg.Config)
    if !ok {
        resp.Diagnostics.AddError(
            "Unexpected Provider Data Type",
            fmt.Sprintf("Expected *transport_tpg.Config, got: %T. "+
                "Please report this issue to the provider developers.", req.ProviderData),
        )
        return
    }

    e.providerConfig = config
}

// Open performs the read operation for the ephemeral resource
func (e *ephemeralSecretManagerSecretVersion) Open(
    ctx context.Context,
    req ephemeral.OpenRequest,
    resp *ephemeral.OpenResponse,
) {
    var model ephemeralSecretManagerSecretVersionModel

    // Read configuration into model
    resp.Diagnostics.Append(req.Config.Get(ctx, &model)...)
    if resp.Diagnostics.HasError() {
        return
    }

    config := e.providerConfig
    userAgent := config.UserAgent

    // Convert Plugin Framework types to core data struct
    coreData := &SecretVersionCoreData{
        Project: model.Project.ValueString(),
        Secret:  model.Secret.ValueString(),
        Version: model.Version.ValueString(),
    }

    coreOpts := &SecretVersionCoreConfig{
        IsSecretDataBase64: model.IsSecretDataBase64.ValueBool(),
    }

    // Call the shared core function
    if err := ReadSecretVersionCore(config, userAgent, coreData, coreOpts); err != nil {
        resp.Diagnostics.AddError(
            "Error Reading Secret Version",
            fmt.Sprintf("Could not read secret version: %s", err),
        )
        return
    }

    // Map core data back to Plugin Framework model
    model.Project = types.StringValue(coreData.ResolvedProject)
    model.Version = types.StringValue(coreData.ResolvedVersion)
    model.Name = types.StringValue(coreData.Name)
    model.SecretData = types.StringValue(coreData.SecretData)
    model.CreateTime = types.StringValue(coreData.CreateTime)
    model.Enabled = types.BoolValue(coreData.Enabled)

    if coreData.DestroyTime != "" {
        model.DestroyTime = types.StringValue(coreData.DestroyTime)
    } else {
        model.DestroyTime = types.StringNull()
    }

    // Set the result
    resp.Diagnostics.Append(resp.Result.Set(ctx, model)...)
}
```

---

### Provider Registration

**Template:** `mmv1/third_party/terraform/fwprovider/generated_ephemeral_resources.go.tmpl` (NEW)

**Generated Output:** `google/fwprovider/generated_ephemeral_resources.go`

```go
// Copyright 2024 Google Inc.
// Code generated by Magic Modules. DO NOT EDIT.

package fwprovider

import (
    "github.com/hashicorp/terraform-plugin-framework/ephemeral"

    "github.com/hashicorp/terraform-provider-google/google/services/secretmanager"
    // ... other product imports as needed
)

// generatedEphemeralResources contains all ephemeral resources generated by Magic Modules
var generatedEphemeralResources = []func() ephemeral.EphemeralResource{
    secretmanager.EphemeralSecretManagerSecretVersion,
    // ... other generated ephemeral resources
}
```

**Template:** `mmv1/third_party/terraform/fwprovider/framework_provider.go.tmpl` (MODIFIED)

```go
// EphemeralResources returns all ephemeral resources for the provider
func (p *FrameworkProvider) EphemeralResources(_ context.Context) []func() ephemeral.EphemeralResource {
    // Start with handwritten ephemeral resources
    resources := []func() ephemeral.EphemeralResource{
        // Handwritten resources (these will eventually be migrated to generated)
        resourcemanager.GoogleEphemeralClientConfig,
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
        resourcemanager.GoogleEphemeralServiceAccountIdToken,
        resourcemanager.GoogleEphemeralServiceAccountJwt,
        resourcemanager.GoogleEphemeralServiceAccountKey,
    }

    // Append generated ephemeral resources
    resources = append(resources, generatedEphemeralResources...)

    return resources
}
```

---

### Generation Code Changes

**File:** `mmv1/provider/terraform.go` (MODIFICATIONS)

```go
// GenerateObject generates all files for a single resource
func (t *Terraform) GenerateObject(object api.Resource, outputFolder, productPath string, generateCode, generateDocs bool) {
    templateData := NewTemplateData(outputFolder, t.TargetVersionName, t.templateFS)

    if !object.IsExcluded() {
        log.Printf("Generating %s resource", object.Name)
        t.GenerateResource(object, *templateData, outputFolder, generateCode, generateDocs)

        if generateCode {
            t.GenerateResourceTests(object, *templateData, outputFolder)
            t.GenerateResourceSweeper(object, *templateData, outputFolder)

            // Data source generation
            t.GenerateSingularDataSource(object, *templateData, outputFolder)
            t.GenerateSingularDataSourceTests(object, *templateData, outputFolder)

            // NEW: Core logic generation (when both data source and ephemeral are enabled)
            t.GenerateDataSourceCore(object, *templateData, outputFolder)

            // NEW: Ephemeral resource generation
            t.GenerateEphemeralResource(object, *templateData, outputFolder)
            t.GenerateEphemeralResourceTests(object, *templateData, outputFolder)

            t.GenerateResourceMetadata(object, *templateData, outputFolder)
        }
    }

    // IAM policy generation...
}

// GenerateDataSourceCore generates the shared core logic file
func (t *Terraform) GenerateDataSourceCore(object api.Resource, templateData TemplateData, outputFolder string) {
    if !object.ShouldGenerateDataSourceCore() {
        return
    }

    productName := t.Product.ApiName
    targetFolder := path.Join(outputFolder, t.FolderName(), "services", productName)
    if err := os.MkdirAll(targetFolder, os.ModePerm); err != nil {
        log.Println(fmt.Errorf("error creating parent directory %v: %v", targetFolder, err))
    }

    targetFilePath := path.Join(targetFolder, fmt.Sprintf("%s_datasource_core.go", t.ResourceGoFilename(object)))
    templateData.GenerateDataSourceCoreFile(targetFilePath, object)
}

// GenerateEphemeralResource generates the Plugin Framework ephemeral resource
func (t *Terraform) GenerateEphemeralResource(object api.Resource, templateData TemplateData, outputFolder string) {
    if !object.ShouldGenerateEphemeralResource() {
        return
    }

    productName := t.Product.ApiName
    targetFolder := path.Join(outputFolder, t.FolderName(), "services", productName)
    if err := os.MkdirAll(targetFolder, os.ModePerm); err != nil {
        log.Println(fmt.Errorf("error creating parent directory %v: %v", targetFolder, err))
    }

    targetFilePath := path.Join(targetFolder, fmt.Sprintf("ephemeral_%s.go", t.ResourceGoFilename(object)))
    templateData.GenerateEphemeralResourceFile(targetFilePath, object)
}

// GenerateEphemeralResourceTests generates tests for the ephemeral resource
func (t *Terraform) GenerateEphemeralResourceTests(object api.Resource, templateData TemplateData, outputFolder string) {
    if !object.ShouldGenerateEphemeralResourceTests() {
        return
    }

    productName := t.Product.ApiName
    targetFolder := path.Join(outputFolder, t.FolderName(), "services", productName)
    if err := os.MkdirAll(targetFolder, os.ModePerm); err != nil {
        log.Println(fmt.Errorf("error creating parent directory %v: %v", targetFolder, err))
    }

    targetFilePath := path.Join(targetFolder, fmt.Sprintf("ephemeral_%s_test.go", t.ResourceGoFilename(object)))
    templateData.GenerateEphemeralResourceTestFile(targetFilePath, object)
}
```

**File:** `mmv1/provider/template_data.go` (ADDITIONS)

```go
// GenerateDataSourceCoreFile generates the shared core logic file
func (td *TemplateData) GenerateDataSourceCoreFile(filePath string, r api.Resource) {
    templatePath := "templates/terraform/datasource_core.go.tmpl"
    templates := []string{templatePath}
    td.GenerateFile(filePath, templatePath, r, true, templates...)
}

// GenerateEphemeralResourceFile generates the Plugin Framework ephemeral resource
func (td *TemplateData) GenerateEphemeralResourceFile(filePath string, r api.Resource) {
    templatePath := "templates/terraform/ephemeral.go.tmpl"
    templates := []string{
        templatePath,
        "templates/terraform/ephemeral_schema.go.tmpl",
    }
    td.GenerateFile(filePath, templatePath, r, true, templates...)
}

// GenerateEphemeralResourceTestFile generates tests for the ephemeral resource
func (td *TemplateData) GenerateEphemeralResourceTestFile(filePath string, r api.Resource) {
    templatePath := "templates/terraform/ephemeral_test.go.tmpl"
    templates := []string{templatePath}
    td.GenerateFile(filePath, templatePath, r, true, templates...)
}
```

---

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          USER TERRAFORM CONFIG                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            │                                                      │
            ▼                                                      ▼
┌─────────────────────────┐                          ┌─────────────────────────┐
│      DATA SOURCE        │                          │   EPHEMERAL RESOURCE    │
│ (terraform-plugin-sdk)  │                          │(terraform-plugin-framework)│
│                         │                          │                         │
│ data "google_secret_    │                          │ ephemeral "google_      │
│   manager_secret_       │                          │   secret_manager_       │
│   version" "example" {  │                          │   secret_version" "ex" {│
│   secret = "my-secret"  │                          │   secret = "my-secret"  │
│ }                       │                          │ }                       │
└─────────────────────────┘                          └─────────────────────────┘
            │                                                      │
            ▼                                                      ▼
┌─────────────────────────┐                          ┌─────────────────────────┐
│ dataSourceSecretManager │                          │ ephemeralSecretManager  │
│ SecretVersionRead()     │                          │ SecretVersion.Open()    │
│                         │                          │                         │
│ Converts schema.        │                          │ Converts Plugin         │
│ ResourceData to         │                          │ Framework model to      │
│ CoreData struct         │                          │ CoreData struct         │
└─────────────────────────┘                          └─────────────────────────┘
            │                                                      │
            └──────────────────────────┬───────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     ReadSecretVersionCore()                                  │
│                 (secret_version_datasource_core.go)                         │
│                                                                             │
│  1. Resolves project from config or provider default                        │
│  2. Builds API URL: projects/{project}/secrets/{secret}/versions/{version}  │
│  3. GET request to fetch version metadata                                   │
│  4. GET request to access secret data                                       │
│  5. Decodes base64 payload (if not requested as base64)                     │
│  6. Populates CoreData struct with results                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Secret Manager API                                       │
│                     (Google Cloud)                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          RESPONSE                                           │
└─────────────────────────────────────────────────────────────────────────────┘
            ┌──────────────────────────┴──────────────────────────┐
            │                                                      │
            ▼                                                      ▼
┌─────────────────────────┐                          ┌─────────────────────────┐
│      DATA SOURCE        │                          │   EPHEMERAL RESOURCE    │
│                         │                          │                         │
│ Maps CoreData back to   │                          │ Maps CoreData back to   │
│ schema.ResourceData     │                          │ Plugin Framework model  │
│                         │                          │                         │
│ Sets d.SetId(name)      │                          │ Sets resp.Result        │
│ Stores in STATE FILE    │                          │ NOT stored anywhere     │
└─────────────────────────┘                          └─────────────────────────┘
```

---

## Other Options (Brief Summary)

### Option B: Ephemeral as Data Source Adapter

Direct runtime wrapping of data source. Complex type conversion, fragile. Not recommended.

### Option C: Dual Generation from Same Definition

Separate templates with duplicated logic. Simpler but more maintenance burden.

### Option D: Plugin Framework Only

Requires migrating all data sources. Too disruptive for existing codebase.

### Option E: Shared API Layer

Clean architecture but requires significant refactoring of existing patterns.

---

## Why Wrapper Pattern is Best

| Criteria | Wrapper Pattern | Alternatives |
|----------|----------------|--------------|
| **Incremental adoption** | Can add to any resource | Requires migration |
| **Code reuse** | Core logic shared | Varies |
| **Framework compatibility** | Both SDK v2 + PF | Limited |
| **Testability** | Core can be unit tested | Framework-dependent |
| **Maintenance** | Change once, affects both | Change in multiple places |
| **Performance** | Native to each framework | Conversion overhead |
| **Debugging** | Clear separation | Intertwined |
