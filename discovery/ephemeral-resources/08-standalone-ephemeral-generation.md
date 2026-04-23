# Standalone Ephemeral Generation (Recommended Approach)

## Context

After team discussion with Cameron, the wrapper pattern (wrapping data sources to create ephemeral resources) was determined to be **blocked** because:

1. Generated data sources delegate to `resourceXxxRead()` (SDK v2)
2. Ephemeral resources use Plugin Framework
3. Can't call SDK v2 code from Plugin Framework
4. Would require full SDK v2 → Plugin Framework migration first

**New Direction:** Generate standalone ephemeral resources with their own API logic.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STANDALONE EPHEMERAL GENERATION                           │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           YAML Resource Definition                          │
│                    (mmv1/products/<product>/<Resource>.yaml)                │
│                                                                             │
│  ephemeral_experimental:                                                    │
│    generate: true                                                           │
│    sensitive_fields:                                                        │
│      - secretData                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            mmv1 Generator                                   │
│                                                                             │
│  GenerateEphemeralResource()                                                │
│    → Uses ephemeral.go.tmpl                                                 │
│    → Generates Plugin Framework code                                        │
│    → Includes API logic directly (not shared)                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Generated: ephemeral_xxx.go                              │
│                                                                             │
│  - Complete Plugin Framework implementation                                 │
│  - Own schema (not derived from resource)                                   │
│  - Own API logic (duplicated from resource, but standalone)                 │
│  - Registered in framework_provider.go                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Difference from Wrapper Pattern:**
- Wrapper Pattern: Ephemeral → Data Source → Resource (blocked by SDK v2)
- Standalone: Ephemeral has its own complete implementation

---

## File Structure

```
mmv1/
├── products/
│   └── secretmanager/
│       └── SecretVersion.yaml              # Add ephemeral_experimental block
│
├── templates/
│   └── terraform/
│       ├── ephemeral.go.tmpl               # NEW: Full ephemeral template
│       └── ephemeral_test.go.tmpl          # NEW: Test template
│
├── api/
│   └── resource/
│       └── ephemeral.go                    # NEW: Ephemeral config struct
│
└── provider/
    └── terraform.go                        # Add GenerateEphemeralResource()

# Generated Output:
google/services/secretmanager/
├── resource_secret_manager_secret_version.go     # Existing (SDK v2)
├── data_source_secret_manager_secret_version.go  # Existing (SDK v2, handwritten)
└── ephemeral_secret_manager_secret_version.go    # NEW (Plugin Framework)
```

---

## YAML Configuration

```yaml
# mmv1/products/secretmanager/SecretVersion.yaml

name: 'SecretVersion'
description: |
  A Secret Version holds the actual secret data.

base_url: 'projects/{{project}}/secrets/{{secret}}/versions'
self_link: 'projects/{{project}}/secrets/{{secret}}/versions/{{version}}'

# Existing resource properties...
properties:
  - name: 'project'
    type: String
    description: 'Project ID'
    url_param_only: true

  - name: 'secret'
    type: String
    description: 'The secret ID'
    required: true
    url_param_only: true

  - name: 'version'
    type: String
    description: 'The version'
    url_param_only: true

  - name: 'secretData'
    type: String
    description: 'The secret data'
    output: true

# ============================================================================
# NEW: Ephemeral Resource Generation
# ============================================================================
ephemeral_experimental:
  generate: true

  # Fields to mark as Sensitive in Plugin Framework schema
  sensitive_fields:
    - 'secretData'

  # Optional: Override description for ephemeral resource
  description: |
    Provides ephemeral access to secret data without storing in state.
```

---

## Generated Code: Ephemeral Resource

**Template:** `mmv1/templates/terraform/ephemeral.go.tmpl`

**Output:** `google/services/secretmanager/ephemeral_secret_manager_secret_version.go`

```go
// Code generated by Magic Modules. DO NOT EDIT.

package secretmanager

import (
    "context"
    "encoding/base64"
    "fmt"
    "regexp"

    "github.com/hashicorp/terraform-plugin-framework/ephemeral"
    "github.com/hashicorp/terraform-plugin-framework/ephemeral/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"

    transport_tpg "github.com/hashicorp/terraform-provider-google/google/transport"
)

var _ ephemeral.EphemeralResource = &ephemeralSecretManagerSecretVersion{}

func EphemeralSecretManagerSecretVersion() ephemeral.EphemeralResource {
    return &ephemeralSecretManagerSecretVersion{}
}

type ephemeralSecretManagerSecretVersion struct {
    providerConfig *transport_tpg.Config
}

// Model - generated from YAML properties
type ephemeralSecretManagerSecretVersionModel struct {
    Project            types.String `tfsdk:"project"`
    Secret             types.String `tfsdk:"secret"`
    Version            types.String `tfsdk:"version"`
    IsSecretDataBase64 types.Bool   `tfsdk:"is_secret_data_base64"`
    Name               types.String `tfsdk:"name"`
    SecretData         types.String `tfsdk:"secret_data"`
    CreateTime         types.String `tfsdk:"create_time"`
    DestroyTime        types.String `tfsdk:"destroy_time"`
    Enabled            types.Bool   `tfsdk:"enabled"`
}

func (e *ephemeralSecretManagerSecretVersion) Metadata(ctx context.Context, req ephemeral.MetadataRequest, resp *ephemeral.MetadataResponse) {
    // Generated from YAML name
    resp.TypeName = req.ProviderTypeName + "_secret_manager_secret_version"
}

func (e *ephemeralSecretManagerSecretVersion) Schema(ctx context.Context, req ephemeral.SchemaRequest, resp *ephemeral.SchemaResponse) {
    // Generated from YAML properties
    resp.Schema = schema.Schema{
        Description: "Provides ephemeral access to secret data without storing in state.",
        Attributes: map[string]schema.Attribute{
            "project": schema.StringAttribute{
                Description: "Project ID",
                Optional:    true,
                Computed:    true,
            },
            "secret": schema.StringAttribute{
                Description: "The secret ID",
                Required:    true,
            },
            "version": schema.StringAttribute{
                Description: "The version",
                Optional:    true,
                Computed:    true,
            },
            "is_secret_data_base64": schema.BoolAttribute{
                Description: "If true, return secret data as base64",
                Optional:    true,
            },
            "name": schema.StringAttribute{
                Description: "Full resource name",
                Computed:    true,
            },
            "secret_data": schema.StringAttribute{
                Description: "The secret data",
                Computed:    true,
                Sensitive:   true,  // <-- From sensitive_fields in YAML
            },
            "create_time": schema.StringAttribute{
                Description: "Creation time",
                Computed:    true,
            },
            "destroy_time": schema.StringAttribute{
                Description: "Destruction time",
                Computed:    true,
            },
            "enabled": schema.BoolAttribute{
                Description: "Whether the version is enabled",
                Computed:    true,
            },
        },
    }
}

func (e *ephemeralSecretManagerSecretVersion) Configure(ctx context.Context, req ephemeral.ConfigureRequest, resp *ephemeral.ConfigureResponse) {
    if req.ProviderData == nil {
        return
    }
    config, ok := req.ProviderData.(*transport_tpg.Config)
    if !ok {
        resp.Diagnostics.AddError("Unexpected provider data type", "")
        return
    }
    e.providerConfig = config
}

func (e *ephemeralSecretManagerSecretVersion) Open(ctx context.Context, req ephemeral.OpenRequest, resp *ephemeral.OpenResponse) {
    var model ephemeralSecretManagerSecretVersionModel
    resp.Diagnostics.Append(req.Config.Get(ctx, &model)...)
    if resp.Diagnostics.HasError() {
        return
    }

    config := e.providerConfig
    userAgent := config.UserAgent

    // =========================================================================
    // API LOGIC - Generated from YAML (standalone, not shared with resource)
    // =========================================================================

    project := model.Project.ValueString()
    if project == "" {
        project = config.Project
    }

    secret := model.Secret.ValueString()
    version := model.Version.ValueString()
    if version == "" {
        version = "latest"
    }

    // Build URL - generated from base_url/self_link
    url := fmt.Sprintf("%sprojects/%s/secrets/%s/versions/%s",
        config.SecretManagerBasePath, project, secret, version)

    // GET version metadata
    versionResp, err := transport_tpg.SendRequest(transport_tpg.SendRequestOptions{
        Config:    config,
        Method:    "GET",
        Project:   project,
        RawURL:    url,
        UserAgent: userAgent,
    })
    if err != nil {
        resp.Diagnostics.AddError("Error reading secret version", err.Error())
        return
    }

    // Parse response
    nameValue, ok := versionResp["name"].(string)
    if !ok {
        resp.Diagnostics.AddError("Invalid response", "Missing 'name' field")
        return
    }

    // Extract version number
    secretVersionRegex := regexp.MustCompile(`projects/(.+)/secrets/(.+)/versions/(.+)$`)
    parts := secretVersionRegex.FindStringSubmatch(nameValue)
    if len(parts) != 4 {
        resp.Diagnostics.AddError("Invalid name format", nameValue)
        return
    }

    // Access secret data
    accessURL := fmt.Sprintf("%s:access", url)
    accessResp, err := transport_tpg.SendRequest(transport_tpg.SendRequestOptions{
        Config:    config,
        Method:    "GET",
        Project:   project,
        RawURL:    accessURL,
        UserAgent: userAgent,
    })
    if err != nil {
        resp.Diagnostics.AddError("Error accessing secret", err.Error())
        return
    }

    payload := accessResp["payload"].(map[string]interface{})
    payloadData := payload["data"].(string)

    if !model.IsSecretDataBase64.ValueBool() {
        decoded, err := base64.StdEncoding.DecodeString(payloadData)
        if err != nil {
            resp.Diagnostics.AddError("Error decoding secret", err.Error())
            return
        }
        payloadData = string(decoded)
    }

    // =========================================================================
    // Set result
    // =========================================================================
    model.Project = types.StringValue(project)
    model.Version = types.StringValue(parts[3])
    model.Name = types.StringValue(nameValue)
    model.SecretData = types.StringValue(payloadData)
    model.Enabled = types.BoolValue(true)

    if createTime, ok := versionResp["createTime"].(string); ok {
        model.CreateTime = types.StringValue(createTime)
    }
    if destroyTime, ok := versionResp["destroyTime"].(string); ok {
        model.DestroyTime = types.StringValue(destroyTime)
    } else {
        model.DestroyTime = types.StringNull()
    }

    resp.Diagnostics.Append(resp.Result.Set(ctx, model)...)
}
```

---

## Template Structure

The template generates:

1. **Struct definition** - From resource name
2. **Model struct** - From YAML properties, with `tfsdk` tags
3. **Metadata()** - Resource type name from YAML
4. **Schema()** - From YAML properties, with sensitive fields marked
5. **Configure()** - Standard provider config setup
6. **Open()** - API logic generated from:
   - `base_url` / `self_link` for URL construction
   - Properties for field mapping
   - Standard transport_tpg patterns

---

## Trade-offs

### Pros

| Benefit | Explanation |
|---------|-------------|
| **No blocker** | Doesn't depend on SDK v2 → PF migration |
| **Clean PF code** | Pure Plugin Framework, no hacks |
| **Independent** | Changes to resource don't affect ephemeral |
| **Can start now** | No prerequisites |

### Cons

| Drawback | Explanation |
|----------|-------------|
| **Code duplication** | API logic duplicated from resource |
| **Maintenance burden** | API changes need updating in 2+ places |
| **Drift risk** | Ephemeral and resource could diverge |
| **More generated code** | Larger codebase |

---

## Future: Reunification After Migration

Once the SDK v2 → Plugin Framework migration is complete for resources, we can revisit the unified approach:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FUTURE STATE (Post-Migration)                             │
└─────────────────────────────────────────────────────────────────────────────┘

  After resources are migrated to Plugin Framework:

  ┌─────────────────────────────────────────────┐
  │  Resource (Plugin Framework)                │
  │                                             │
  │  func (r *resource) Read() {                │
  │      readSecretVersionCore(...)  ◄──────────┼─┐
  │  }                                          │ │
  └─────────────────────────────────────────────┘ │
                                                  │ Can share!
  ┌─────────────────────────────────────────────┐ │
  │  Data Source (Plugin Framework)             │ │
  │                                             │ │
  │  func (d *datasource) Read() {              │ │
  │      readSecretVersionCore(...)  ◄──────────┼─┤
  │  }                                          │ │
  └─────────────────────────────────────────────┘ │
                                                  │
  ┌─────────────────────────────────────────────┐ │
  │  Ephemeral (Plugin Framework)               │ │
  │                                             │ │
  │  func (e *ephemeral) Open() {               │ │
  │      readSecretVersionCore(...)  ◄──────────┼─┘
  │  }                                          │
  └─────────────────────────────────────────────┘

  All three can share the same core function!
```

For now, standalone generation is the practical path forward.

---

## Implementation Checklist

- [ ] Create `mmv1/api/resource/ephemeral.go` struct
- [ ] Add `Ephemeral` field to `Resource` struct
- [ ] Create `mmv1/templates/terraform/ephemeral.go.tmpl`
- [ ] Add `GenerateEphemeralResource()` to `terraform.go`
- [ ] Add `GenerateEphemeralResourceFile()` to `template_data.go`
- [ ] Update `framework_provider.go.tmpl` to include generated ephemeral resources
- [ ] Create `ephemeral_test.go.tmpl` for test generation
- [ ] Test with a simple resource first (e.g., SecretVersion)
- [ ] Document YAML configuration options
