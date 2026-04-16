# Current Ephemeral Resources Implementation

## Overview

All ephemeral resources in terraform-provider-google are currently **handwritten** and located in `mmv1/third_party/terraform/services/`. They use the Terraform Plugin Framework (not SDK v2).

## Existing Ephemeral Resources

| Resource Name | File Location | Corresponding Data Source |
|--------------|---------------|---------------------------|
| `google_client_config` | `resourcemanager/ephemeral_google_client_config.go` | `google_client_config` (handwritten) |
| `google_service_account_access_token` | `resourcemanager/ephemeral_google_service_account_access_token.go` | `google_service_account_access_token` (handwritten) |
| `google_service_account_id_token` | `resourcemanager/ephemeral_google_service_account_id_token.go` | `google_service_account_id_token` (handwritten) |
| `google_service_account_jwt` | `resourcemanager/ephemeral_google_service_account_jwt.go` | `google_service_account_jwt` (handwritten) |
| `google_service_account_key` | `resourcemanager/ephemeral_google_service_account_key.go` | `google_service_account_key` (handwritten) |
| `google_secret_manager_secret_version` | `secretmanager/ephemeral_google_secret_manager_secret_version.go` | `google_secret_manager_secret_version` (handwritten) |

## Code Structure Analysis

### Example: Secret Manager Secret Version

**Ephemeral Resource:** `ephemeral_google_secret_manager_secret_version.go`

```go
package secretmanager

import (
    "context"
    "github.com/hashicorp/terraform-plugin-framework/ephemeral"
    "github.com/hashicorp/terraform-plugin-framework/ephemeral/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"
    transport_tpg "github.com/hashicorp/terraform-provider-google/google/transport"
)

// Interface compliance
var _ ephemeral.EphemeralResource = &googleEphemeralSecretManagerSecretVersion{}

// Constructor
func GoogleEphemeralSecretManagerSecretVersion() ephemeral.EphemeralResource {
    return &googleEphemeralSecretManagerSecretVersion{}
}

// Struct definition
type googleEphemeralSecretManagerSecretVersion struct {
    providerConfig *transport_tpg.Config
}

// Model for state
type ephemeralSecretManagerSecretVersionModel struct {
    Project            types.String `tfsdk:"project"`
    Secret             types.String `tfsdk:"secret"`
    Version            types.String `tfsdk:"version"`
    IsSecretDataBase64 types.Bool   `tfsdk:"is_secret_data_base64"`
    SecretData         types.String `tfsdk:"secret_data"`
    Name               types.String `tfsdk:"name"`
    CreateTime         types.String `tfsdk:"create_time"`
    DestroyTime        types.String `tfsdk:"destroy_time"`
    Enabled            types.Bool   `tfsdk:"enabled"`
}

// Required methods
func (p *googleEphemeralSecretManagerSecretVersion) Metadata(ctx context.Context, req ephemeral.MetadataRequest, resp *ephemeral.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_secret_manager_secret_version"
}

func (p *googleEphemeralSecretManagerSecretVersion) Schema(ctx context.Context, req ephemeral.SchemaRequest, resp *ephemeral.SchemaResponse) {
    resp.Schema = schema.Schema{
        Attributes: map[string]schema.Attribute{
            "project": schema.StringAttribute{
                Description: "The project to get the secret version for.",
                Optional:    true,
                Computed:    true,
            },
            "secret": schema.StringAttribute{
                Description: "The secret to get the secret version for.",
                Required:    true,
            },
            "secret_data": schema.StringAttribute{
                Description: "The secret data.",
                Computed:    true,
                Sensitive:   true,  // <-- KEY DIFFERENCE
            },
            // ... other attributes
        },
    }
}

func (p *googleEphemeralSecretManagerSecretVersion) Configure(ctx context.Context, req ephemeral.ConfigureRequest, resp *ephemeral.ConfigureResponse) {
    // Get provider config from ProviderData
    pd, ok := req.ProviderData.(*transport_tpg.Config)
    p.providerConfig = pd
}

func (p *googleEphemeralSecretManagerSecretVersion) Open(ctx context.Context, req ephemeral.OpenRequest, resp *ephemeral.OpenResponse) {
    var data ephemeralSecretManagerSecretVersionModel
    resp.Diagnostics.Append(req.Config.Get(ctx, &data)...)

    // Make API calls to fetch data
    // Set result
    resp.Diagnostics.Append(resp.Result.Set(ctx, data)...)
}
```

**Corresponding Data Source:** `data_source_secret_manager_secret_version.go`

```go
package secretmanager

import (
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
    "github.com/hashicorp/terraform-provider-google/google/registry"
)

func DataSourceSecretManagerSecretVersion() *schema.Resource {
    return &schema.Resource{
        Read: dataSourceSecretManagerSecretVersionRead,
        Schema: map[string]*schema.Schema{
            "project": {
                Type:     schema.TypeString,
                Optional: true,
                Computed: true,
            },
            "secret": {
                Type:     schema.TypeString,
                Required: true,
            },
            "secret_data": {
                Type:      schema.TypeString,
                Computed:  true,
                Sensitive: true,
            },
            // ... other fields
        },
    }
}

func dataSourceSecretManagerSecretVersionRead(d *schema.ResourceData, meta interface{}) error {
    // API calls, data processing
    // Similar logic to ephemeral resource
}

func init() {
    registry.Schema{
        Name:        "google_secret_manager_secret_version",
        ProductName: "secretmanager",
        Type:        registry.SchemaTypeDataSource,
        Schema:      DataSourceSecretManagerSecretVersion(),
    }.Register()
}
```

## Key Differences: Ephemeral vs Data Source

| Aspect | Data Source (SDK v2) | Ephemeral Resource (Plugin Framework) |
|--------|---------------------|--------------------------------------|
| **Package** | `terraform-plugin-sdk/v2/helper/schema` | `terraform-plugin-framework/ephemeral` |
| **Schema Type** | `*schema.Resource` | `schema.Schema` |
| **Type System** | Go primitives | `types.String`, `types.Bool`, etc. |
| **State Model** | Uses `*schema.ResourceData` | Uses typed struct with `tfsdk` tags |
| **Read Method** | `Read(d, meta) error` | `Open(ctx, req, resp)` |
| **Registration** | Via `registry.Schema{}.Register()` | Via `provider.EphemeralResources()` |
| **Sensitive** | `Sensitive: true` | `Sensitive: true` |
| **Context** | Implicit | Explicit `context.Context` |

## Provider Registration

**File:** `mmv1/third_party/terraform/fwprovider/framework_provider.go.tmpl`

```go
// Interface compliance
var (
    _ provider.ProviderWithEphemeralResources = &FrameworkProvider{}
)

// Configure passes provider config to ephemeral resources
func (p *FrameworkProvider) Configure(ctx context.Context, req provider.ConfigureRequest, resp *provider.ConfigureResponse) {
    meta := p.Primary.Meta().(*transport_tpg.Config)
    resp.EphemeralResourceData = meta  // Passed to Configure()
}

// EphemeralResources returns all ephemeral resources
func (p *FrameworkProvider) EphemeralResources(_ context.Context) []func() ephemeral.EphemeralResource {
    return []func() ephemeral.EphemeralResource{
        resourcemanager.GoogleEphemeralClientConfig,
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
        resourcemanager.GoogleEphemeralServiceAccountIdToken,
        resourcemanager.GoogleEphemeralServiceAccountJwt,
        resourcemanager.GoogleEphemeralServiceAccountKey,
        secretmanager.GoogleEphemeralSecretManagerSecretVersion,
    }
}
```

## Required Interface Methods

Ephemeral resources must implement `ephemeral.EphemeralResource`:

```go
type EphemeralResource interface {
    // Required
    Metadata(context.Context, MetadataRequest, *MetadataResponse)
    Schema(context.Context, SchemaRequest, *SchemaResponse)
    Open(context.Context, OpenRequest, *OpenResponse)
}

// Optional interfaces
type EphemeralResourceWithConfigure interface {
    Configure(context.Context, ConfigureRequest, *ConfigureResponse)
}

type EphemeralResourceWithRenew interface {
    Renew(context.Context, RenewRequest, *RenewResponse)
}

type EphemeralResourceWithClose interface {
    Close(context.Context, CloseRequest, *CloseResponse)
}
```

## Code Duplication Analysis

Comparing `ephemeral_google_secret_manager_secret_version.go` with `data_source_secret_manager_secret_version.go`:

**Shared Logic (~70%):**
- Project resolution
- URL construction
- API calls to get version metadata
- API calls to access secret data
- Base64 decoding logic
- Field mapping

**Different Logic (~30%):**
- Schema definition (different framework)
- State handling (typed model vs ResourceData)
- Registration mechanism
- Error handling style

## Implications for Unified Generation

1. **Cannot reuse SDK v2 Read function directly** - Need to generate Plugin Framework code
2. **Model struct generation required** - Need to generate typed model with `tfsdk` tags
3. **Schema conversion** - Must generate Plugin Framework schema from same definition
4. **Sensitive field marking** - Need to identify and mark sensitive fields
5. **Provider registration** - Need to generate entries in `EphemeralResources()` method

## Potential Shared Components

Components that could be shared between generated data sources and ephemeral resources:

1. **API client calls** - Same transport layer
2. **URL construction** - Same base URLs
3. **Field extraction** - Same IdFormat parsing
4. **Validation logic** - Could share validators
5. **Description text** - Same documentation

## Documentation

Ephemeral resource documentation lives in:
```
mmv1/third_party/terraform/website/docs/ephemeral-resources/
```

Example files:
- `service_account_access_token.html.markdown`
- `service_account_id_token.html.markdown`
- `secret_manager_secret_version.html.markdown`
