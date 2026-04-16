# Key Code References

## Quick Reference: File Locations

### Generation Pipeline

| File | Purpose |
|------|---------|
| `mmv1/cmd/main.go` | Entry point for generation |
| `mmv1/loader/loader.go` | Loads products and resources from YAML |
| `mmv1/api/resource.go` | Resource struct definition (~2,666 lines) |
| `mmv1/api/resource/datasource.go` | Datasource configuration struct |
| `mmv1/provider/terraform.go` | Terraform provider generation logic |
| `mmv1/provider/template_data.go` | Template execution engine |

### Templates

| File | Purpose |
|------|---------|
| `mmv1/templates/terraform/resource.go.tmpl` | Main resource template |
| `mmv1/templates/terraform/datasource.go.tmpl` | Data source template |
| `mmv1/templates/terraform/datasource_test_file.go.tmpl` | Data source test template |

### Existing Ephemeral Resources

| File | Resource |
|------|----------|
| `mmv1/third_party/terraform/services/resourcemanager/ephemeral_google_client_config.go` | Client config |
| `mmv1/third_party/terraform/services/resourcemanager/ephemeral_google_service_account_access_token.go` | Access token |
| `mmv1/third_party/terraform/services/resourcemanager/ephemeral_google_service_account_id_token.go` | ID token |
| `mmv1/third_party/terraform/services/resourcemanager/ephemeral_google_service_account_jwt.go` | JWT |
| `mmv1/third_party/terraform/services/resourcemanager/ephemeral_google_service_account_key.go` | Service account key |
| `mmv1/third_party/terraform/services/secretmanager/ephemeral_google_secret_manager_secret_version.go` | Secret version |

### Provider Registration

| File | Purpose |
|------|---------|
| `mmv1/third_party/terraform/fwprovider/framework_provider.go.tmpl` | Plugin Framework provider |
| `mmv1/third_party/terraform/registry/registry.go` | SDK v2 registry system |

### Data Sources with Ephemeral Equivalents

| Data Source | Ephemeral Resource | Notes |
|-------------|-------------------|-------|
| `data_source_secret_manager_secret_version.go` | `ephemeral_google_secret_manager_secret_version.go` | Nearly identical |
| `data_source_google_service_account_access_token.go` | `ephemeral_google_service_account_access_token.go` | Nearly identical |

---

## Code Snippets

### How Data Sources Are Generated

**Entry Point:** `mmv1/provider/terraform.go:112`
```go
t.GenerateSingularDataSource(object, *templateData, outputFolder)
```

**Generation Check:** `mmv1/api/resource.go:2441`
```go
func (r *Resource) ShouldGenerateSingularDataSource() bool {
    if r.Datasource == nil {
        return false
    }
    return r.Datasource.Generate
}
```

**File Generation:** `mmv1/provider/terraform.go:285`
```go
func (t *Terraform) GenerateSingularDataSource(object api.Resource, templateData TemplateData, outputFolder string) {
    if !object.ShouldGenerateSingularDataSource() {
        return
    }
    // ...
    targetFilePath := path.Join(targetFolder, fmt.Sprintf("data_source_%s.go", t.ResourceGoFilename(object)))
    templateData.GenerateDataSourceFile(targetFilePath, object)
}
```

### How Ephemeral Resources Are Registered

**Provider Method:** `mmv1/third_party/terraform/fwprovider/framework_provider.go.tmpl:386`
```go
func (p *FrameworkProvider) EphemeralResources(_ context.Context) []func() ephemeral.EphemeralResource {
    return []func() ephemeral.EphemeralResource{
        resourcemanager.GoogleEphemeralClientConfig,
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
        // ...
    }
}
```

### Ephemeral Resource Structure Pattern

**Interface Compliance:**
```go
var _ ephemeral.EphemeralResource = &googleEphemeralSecretManagerSecretVersion{}
```

**Constructor:**
```go
func GoogleEphemeralSecretManagerSecretVersion() ephemeral.EphemeralResource {
    return &googleEphemeralSecretManagerSecretVersion{}
}
```

**Required Methods:**
- `Metadata(ctx, req, resp)` - Returns type name
- `Schema(ctx, req, resp)` - Returns schema definition
- `Open(ctx, req, resp)` - Performs the read operation

**Optional Methods:**
- `Configure(ctx, req, resp)` - Receives provider config
- `Renew(ctx, req, resp)` - Renews expiring resources
- `Close(ctx, req, resp)` - Cleanup on resource close

---

## YAML Configuration Examples

### Data Source with Ephemeral Support (Proposed)

```yaml
# mmv1/products/secretmanager/SecretVersion.yaml
name: 'SecretVersion'
description: |
  A secret version resource that holds the actual secret data.
base_url: 'projects/{{project}}/secrets/{{secret}}/versions'
id_format: 'projects/{{project}}/secrets/{{secret}}/versions/{{version}}'

# Existing data source config
datasource_experimental:
  generate: true
  exclude_test: false

# NEW: Ephemeral resource config
ephemeral_experimental:
  generate: true
  exclude_test: false
  sensitive_fields:
    - secretData

properties:
  - name: 'project'
    type: String
    description: 'The project containing the secret.'
    url_param_only: true

  - name: 'secret'
    type: String
    description: 'The secret to retrieve.'
    required: true
    url_param_only: true

  - name: 'version'
    type: String
    description: 'The version to retrieve.'
    url_param_only: true

  - name: 'secretData'
    type: String
    description: 'The secret data.'
    output: true
    # Will be marked sensitive in ephemeral resource
```

### Current Data Source-Enabled Resource

```yaml
# mmv1/products/cloudrun/Service.yaml (line 33)
datasource_experimental:
  generate: true
  exclude_test: true
```

---

## Template Function Reference

**Available in templates:** `mmv1/google/template_utils.go`

| Function | Purpose |
|----------|---------|
| `camelize(s)` | Convert to CamelCase |
| `underscore(s)` | Convert to snake_case |
| `lower(s)` | Lowercase |
| `upper(s)` | Uppercase |
| `title(s)` | Title case |
| `customTemplate(obj, path)` | Load custom template |
| `contains(s, substr)` | String contains check |
| `hasPrefix(s, prefix)` | Prefix check |
| `hasSuffix(s, suffix)` | Suffix check |

---

## Build Commands

```bash
# Generate for a specific product
OUTPUT_PATH=/path/to/terraform-provider-google VERSION=ga PRODUCT=secretmanager make build

# Generate for a specific resource
OUTPUT_PATH=/path/to/terraform-provider-google VERSION=ga PRODUCT=secretmanager RESOURCE=SecretVersion make build

# Run tests
make test
```

---

## Key Interfaces

### Ephemeral Resource Interface

```go
// From terraform-plugin-framework/ephemeral
type EphemeralResource interface {
    Metadata(context.Context, MetadataRequest, *MetadataResponse)
    Schema(context.Context, SchemaRequest, *SchemaResponse)
    Open(context.Context, OpenRequest, *OpenResponse)
}

type EphemeralResourceWithConfigure interface {
    EphemeralResource
    Configure(context.Context, ConfigureRequest, *ConfigureResponse)
}
```

### Provider Interface

```go
// From terraform-plugin-framework/provider
type ProviderWithEphemeralResources interface {
    Provider
    EphemeralResources(context.Context) []func() ephemeral.EphemeralResource
}
```

---

## Related Documentation

### Terraform Plugin Framework

- [Ephemeral Resources Guide](https://developer.hashicorp.com/terraform/plugin/framework/ephemeral-resources)
- [Schema Definition](https://developer.hashicorp.com/terraform/plugin/framework/handling-data/schemas)
- [Types System](https://developer.hashicorp.com/terraform/plugin/framework/handling-data/types)

### terraform-provider-google

- [Using Ephemeral Resources](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/using_ephemeral_resources)
- [Provider v7.0 Release Notes](https://github.com/hashicorp/terraform-provider-google/releases/tag/v7.0.0)

### Magic Modules

- [CLAUDE.md](../../CLAUDE.md) - Repository overview
- [Contributing Guide](https://googlecloudplatform.github.io/magic-modules/)
