# Implementation Plan

## Phase 1: Foundation (Week 1-2)

### 1.1 YAML Schema Extension

**File:** `mmv1/api/resource/datasource.go`

Extend the Datasource struct to support ephemeral generation:

```go
type Datasource struct {
    Generate    bool `yaml:"generate"`
    ExcludeTest bool `yaml:"exclude_test"`

    // NEW: Ephemeral resource generation
    GenerateEphemeral    bool     `yaml:"generate_ephemeral"`
    EphemeralExcludeTest bool     `yaml:"ephemeral_exclude_test"`
    SensitiveFields      []string `yaml:"sensitive_fields"`
}
```

**Alternative:** Create a separate `Ephemeral` struct:

```go
// mmv1/api/resource/ephemeral.go
type Ephemeral struct {
    Generate        bool     `yaml:"generate"`
    ExcludeTest     bool     `yaml:"exclude_test"`
    SensitiveFields []string `yaml:"sensitive_fields"`
}
```

### 1.2 Resource Struct Updates

**File:** `mmv1/api/resource.go`

Add method to check for ephemeral generation:

```go
func (r *Resource) ShouldGenerateEphemeralResource() bool {
    if r.Datasource == nil {
        return false
    }
    return r.Datasource.GenerateEphemeral
}

func (r *Resource) ShouldGenerateEphemeralResourceTests() bool {
    if r.Datasource == nil {
        return false
    }
    return r.Datasource.GenerateEphemeral && !r.Datasource.EphemeralExcludeTest
}

func (r *Resource) EphemeralSensitiveFields() []string {
    if r.Datasource == nil {
        return []string{}
    }
    return r.Datasource.SensitiveFields
}
```

### 1.3 Provider Generation Updates

**File:** `mmv1/provider/terraform.go`

Add ephemeral resource generation to the pipeline:

```go
func (t *Terraform) GenerateObject(object api.Resource, ...) {
    // ... existing code ...

    if generateCode {
        // Existing
        t.GenerateSingularDataSource(object, *templateData, outputFolder)
        t.GenerateSingularDataSourceTests(object, *templateData, outputFolder)

        // NEW
        t.GenerateEphemeralResource(object, *templateData, outputFolder)
        t.GenerateEphemeralResourceTests(object, *templateData, outputFolder)
    }
}

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
```

---

## Phase 2: Template Development (Week 2-3)

### 2.1 Create Ephemeral Resource Template

**File:** `mmv1/templates/terraform/ephemeral.go.tmpl`

```go
{{$.CodeHeader TemplatePath}}
package {{ lower $.ProductMetadata.Name }}

import (
    "context"
    "fmt"

    "github.com/hashicorp/terraform-plugin-framework/ephemeral"
    "github.com/hashicorp/terraform-plugin-framework/ephemeral/schema"
    "github.com/hashicorp/terraform-plugin-framework/types"

    transport_tpg "{{ $.ImportPath }}/transport"
    "{{ $.ImportPath }}/tpgresource"
)

var _ ephemeral.EphemeralResource = &ephemeral{{ $.ResourceName }}{}

func Ephemeral{{ $.ResourceName }}() ephemeral.EphemeralResource {
    return &ephemeral{{ $.ResourceName }}{}
}

type ephemeral{{ $.ResourceName }} struct {
    providerConfig *transport_tpg.Config
}

// Model
type ephemeral{{ $.ResourceName }}Model struct {
{{- range $prop := $.Properties }}
    {{ $prop.GoName }} types.{{ $prop.PluginFrameworkType }} `tfsdk:"{{ $prop.Name | underscore }}"`
{{- end }}
}

func (e *ephemeral{{ $.ResourceName }}) Metadata(ctx context.Context, req ephemeral.MetadataRequest, resp *ephemeral.MetadataResponse) {
    resp.TypeName = req.ProviderTypeName + "_{{ $.TerraformResourceName }}"
}

func (e *ephemeral{{ $.ResourceName }}) Schema(ctx context.Context, req ephemeral.SchemaRequest, resp *ephemeral.SchemaResponse) {
    resp.Schema = schema.Schema{
        Description: "{{ $.Description }}",
        Attributes: map[string]schema.Attribute{
{{- range $prop := $.Properties }}
            "{{ $prop.Name | underscore }}": {{ $prop.PluginFrameworkSchemaAttribute }},
{{- end }}
        },
    }
}

func (e *ephemeral{{ $.ResourceName }}) Configure(ctx context.Context, req ephemeral.ConfigureRequest, resp *ephemeral.ConfigureResponse) {
    if req.ProviderData == nil {
        return
    }

    pd, ok := req.ProviderData.(*transport_tpg.Config)
    if !ok {
        resp.Diagnostics.AddError(
            "Unexpected Provider Data",
            fmt.Sprintf("Expected *transport_tpg.Config, got: %T", req.ProviderData),
        )
        return
    }
    e.providerConfig = pd
}

func (e *ephemeral{{ $.ResourceName }}) Open(ctx context.Context, req ephemeral.OpenRequest, resp *ephemeral.OpenResponse) {
    var data ephemeral{{ $.ResourceName }}Model
    resp.Diagnostics.Append(req.Config.Get(ctx, &data)...)
    if resp.Diagnostics.HasError() {
        return
    }

    config := e.providerConfig
    userAgent := config.UserAgent

    // Build URL
    url, err := tpgresource.ReplaceVars{{if $.LegacyLongFormProject}}ForId{{end}}(&data, config, "{{ $.SelfLinkUrl }}")
    if err != nil {
        resp.Diagnostics.AddError("Error building URL", err.Error())
        return
    }

    // Make API call
    res, err := transport_tpg.SendRequest(transport_tpg.SendRequestOptions{
        Config:    config,
        Method:    "GET",
        RawURL:    url,
        UserAgent: userAgent,
    })
    if err != nil {
        resp.Diagnostics.AddError("Error reading resource", err.Error())
        return
    }

    // Set fields from response
{{- range $prop := $.Properties }}
{{- if $prop.Output }}
    if v, ok := res["{{ $prop.ApiName }}"]; ok {
        data.{{ $prop.GoName }} = {{ $prop.PluginFrameworkTypeConversion "v" }}
    }
{{- end }}
{{- end }}

    resp.Diagnostics.Append(resp.Result.Set(ctx, data)...)
}
```

### 2.2 Template Data Extensions

**File:** `mmv1/provider/template_data.go`

Add method to generate ephemeral resource files:

```go
func (td *TemplateData) GenerateEphemeralResourceFile(filePath string, r api.Resource) {
    templatePath := "templates/terraform/ephemeral.go.tmpl"
    templates := []string{
        templatePath,
        "templates/terraform/ephemeral_schema_attribute.go.tmpl",
    }
    td.GenerateFile(filePath, templatePath, r, true, templates...)
}
```

### 2.3 Schema Attribute Generation

**File:** `mmv1/templates/terraform/ephemeral_schema_attribute.go.tmpl`

Helper template for generating Plugin Framework schema attributes:

```go
{{- define "ephemeralSchemaAttribute" -}}
{{- if eq .Type "String" -}}
schema.StringAttribute{
    Description: "{{ .Description }}",
    {{- if .Required }}
    Required: true,
    {{- else if .Output }}
    Computed: true,
    {{- else }}
    Optional: true,
    {{- end }}
    {{- if .IsSensitive }}
    Sensitive: true,
    {{- end }}
}
{{- else if eq .Type "Integer" -}}
schema.Int64Attribute{
    Description: "{{ .Description }}",
    {{- if .Required }}
    Required: true,
    {{- else if .Output }}
    Computed: true,
    {{- else }}
    Optional: true,
    {{- end }}
}
{{- else if eq .Type "Boolean" -}}
schema.BoolAttribute{
    Description: "{{ .Description }}",
    {{- if .Required }}
    Required: true,
    {{- else if .Output }}
    Computed: true,
    {{- else }}
    Optional: true,
    {{- end }}
}
{{- end -}}
{{- end -}}
```

---

## Phase 3: Provider Integration (Week 3-4)

### 3.1 Generated Registration File

Create a template that generates a list of ephemeral resources:

**File:** `mmv1/templates/terraform/ephemeral_resources.go.tmpl`

```go
{{$.CodeHeader TemplatePath}}
package fwprovider

import (
    "github.com/hashicorp/terraform-plugin-framework/ephemeral"
{{- range $product := $.Products }}
{{- if $product.HasEphemeralResources }}
    "{{ $.ImportPath }}/services/{{ $product.Name | lower }}"
{{- end }}
{{- end }}
)

var generatedEphemeralResources = []func() ephemeral.EphemeralResource{
{{- range $product := $.Products }}
{{- range $resource := $product.EphemeralResources }}
    {{ $product.Name | lower }}.Ephemeral{{ $resource.ResourceName }},
{{- end }}
{{- end }}
}
```

### 3.2 Update Framework Provider Template

**File:** `mmv1/third_party/terraform/fwprovider/framework_provider.go.tmpl`

```go
func (p *FrameworkProvider) EphemeralResources(_ context.Context) []func() ephemeral.EphemeralResource {
    resources := []func() ephemeral.EphemeralResource{
        // Handwritten ephemeral resources
        resourcemanager.GoogleEphemeralClientConfig,
        resourcemanager.GoogleEphemeralServiceAccountAccessToken,
        resourcemanager.GoogleEphemeralServiceAccountIdToken,
        resourcemanager.GoogleEphemeralServiceAccountJwt,
        resourcemanager.GoogleEphemeralServiceAccountKey,
        secretmanager.GoogleEphemeralSecretManagerSecretVersion,
    }
    // Append generated ephemeral resources
    resources = append(resources, generatedEphemeralResources...)
    return resources
}
```

---

## Phase 4: Testing Infrastructure (Week 4-5)

### 4.1 Test Template

**File:** `mmv1/templates/terraform/ephemeral_test.go.tmpl`

```go
{{$.CodeHeader TemplatePath}}
package {{ lower $.ProductMetadata.Name }}_test

import (
    "testing"

    "github.com/hashicorp/terraform-plugin-testing/helper/resource"
    "github.com/hashicorp/terraform-provider-google/google/acctest"
)

func TestAccEphemeral{{ $.ResourceName }}_basic(t *testing.T) {
    t.Parallel()

    context := map[string]interface{}{
        "random_suffix": acctest.RandString(t, 10),
    }

    acctest.VcrTest(t, resource.TestCase{
        PreCheck:                 func() { acctest.AccTestPreCheck(t) },
        ProtoV5ProviderFactories: acctest.ProtoV5ProviderFactories(t),
        Steps: []resource.TestStep{
            {
                Config: testAccEphemeral{{ $.ResourceName }}_basic(context),
            },
        },
    })
}

func testAccEphemeral{{ $.ResourceName }}_basic(context map[string]interface{}) string {
    return acctest.Nprintf(`
{{ $.TestConfig }}
`, context)
}
```

### 4.2 Test Configuration Generation

Extend examples to support ephemeral resource testing:

```yaml
examples:
  - name: 'resource_basic'
    primary_resource_id: 'default'
    vars:
      resource_name: 'my-resource'
    ephemeral_test_config: |  # NEW
      ephemeral "google_{{ resource_name }}" "test" {
        name = google_{{ resource_name }}.default.name
      }
```

---

## Phase 5: Documentation (Week 5-6)

### 5.1 Documentation Template

**File:** `mmv1/templates/terraform/website/docs/ephemeral-resources/resource.html.markdown.tmpl`

```markdown
---
subcategory: "{{ $.ProductMetadata.DisplayName }}"
description: |-
  {{ $.Description }}
---

# ephemeral_google_{{ $.TerraformResourceName }}

{{ $.Description }}

~> **Note:** This is an ephemeral resource. Its data is never stored in Terraform state.

## Example Usage

{{ $.ExampleUsage }}

## Argument Reference

{{ $.ArgumentReference }}

## Attributes Reference

{{ $.AttributesReference }}
```

### 5.2 Generate Documentation

Add documentation generation to the pipeline:

```go
func (t *Terraform) GenerateEphemeralResource(object api.Resource, templateData TemplateData, outputFolder string) {
    // ... generate code ...

    // Generate documentation
    docsFolder := path.Join(outputFolder, "website", "docs", "ephemeral-resources")
    docsFilePath := path.Join(docsFolder, fmt.Sprintf("%s.html.markdown", t.ResourceGoFilename(object)))
    templateData.GenerateEphemeralDocumentationFile(docsFilePath, object)
}
```

---

## Phase 6: Migration of Existing Resources (Week 6-8)

### 6.1 Pilot Resource

Start with a simple resource to validate the approach:

```yaml
# mmv1/products/secretmanager/SecretVersion.yaml
name: 'SecretVersion'
# ... existing config ...

datasource_experimental:
  generate: true
  generate_ephemeral: true
  sensitive_fields:
    - secret_data
```

### 6.2 Migration Checklist

For each existing handwritten ephemeral resource:

1. [ ] Add YAML configuration
2. [ ] Verify generated code matches handwritten
3. [ ] Run existing tests against generated code
4. [ ] Update any custom logic
5. [ ] Deprecate handwritten version
6. [ ] Remove handwritten code

### 6.3 Resources to Migrate

| Resource | Complexity | Notes |
|----------|------------|-------|
| `google_secret_manager_secret_version` | Medium | Good pilot candidate |
| `google_client_config` | Low | Special case - provider config |
| `google_service_account_access_token` | Medium | IAM credentials API |
| `google_service_account_id_token` | Medium | IAM credentials API |
| `google_service_account_jwt` | Medium | IAM credentials API |
| `google_service_account_key` | Medium | IAM credentials API |

---

## Success Metrics

1. **Code Generation**
   - Generated ephemeral resources compile without errors
   - Generated code follows Plugin Framework best practices

2. **Functionality**
   - Generated ephemeral resources pass existing tests
   - No regression in data source functionality

3. **Maintainability**
   - Single YAML definition produces both data source and ephemeral resource
   - Template changes apply to all generated resources

4. **Documentation**
   - Generated documentation matches handwritten quality
   - Clear configuration options documented

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Plugin Framework API changes | Pin to stable version, test regularly |
| Performance regression | Benchmark generated vs handwritten |
| Breaking changes to existing resources | Extensive testing, gradual rollout |
| Complex nested schemas | Start with simple resources, iterate |
| Custom code requirements | Allow custom code injection points |

---

## Timeline Summary

| Phase | Duration | Deliverables |
|-------|----------|-------------|
| Phase 1: Foundation | 2 weeks | YAML schema, resource methods |
| Phase 2: Templates | 2 weeks | Ephemeral template, schema generation |
| Phase 3: Integration | 1 week | Provider registration |
| Phase 4: Testing | 1 week | Test infrastructure |
| Phase 5: Documentation | 1 week | Doc templates |
| Phase 6: Migration | 2 weeks | Pilot + existing resources |

**Total: 8-9 weeks**
