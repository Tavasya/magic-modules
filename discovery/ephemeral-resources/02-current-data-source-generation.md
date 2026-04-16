# Current Data Source Generation in mmv1

## Overview

Magic Modules generates data sources through an "experimental" feature that creates a read-only wrapper around existing resource definitions.

## YAML Configuration

Data source generation is controlled via the `datasource_experimental` block in resource YAML files:

```yaml
# mmv1/products/<product>/<Resource>.yaml
name: 'ResourceName'
description: 'Resource description'
base_url: 'projects/{{project}}/resources'
id_format: 'projects/{{project}}/resources/{{name}}'

datasource_experimental:
  generate: true        # Enable data source generation
  exclude_test: false   # Generate tests (default)
```

### Resources Currently Using This

```
mmv1/products/cloudrun/Service.yaml
mmv1/products/iap/Client.yaml
mmv1/products/storagecontrol/FolderIntelligenceConfig.yaml
mmv1/products/storagecontrol/OrganizationIntelligenceConfig.yaml
mmv1/products/storagecontrol/ProjectIntelligenceConfig.yaml
mmv1/products/vmwareengine/Datastore.yaml
```

## YAML Configuration Struct

**File:** `mmv1/api/resource/datasource.go`

```go
type Datasource struct {
    // boolean to determine whether the datasource file should be generated
    Generate bool `yaml:"generate"`
    // boolean to determine whether tests should be generated for a datasource
    ExcludeTest bool `yaml:"exclude_test"`
}
```

## Generation Pipeline

### Entry Point

**File:** `mmv1/provider/terraform.go`

```go
func (t *Terraform) GenerateObject(object api.Resource, outputFolder, productPath string, generateCode, generateDocs bool) {
    // ...
    if generateCode {
        t.GenerateSingularDataSource(object, *templateData, outputFolder)
        t.GenerateSingularDataSourceTests(object, *templateData, outputFolder)
    }
}
```

### Generation Function

```go
func (t *Terraform) GenerateSingularDataSource(object api.Resource, templateData TemplateData, outputFolder string) {
    if !object.ShouldGenerateSingularDataSource() {
        return
    }

    productName := t.Product.ApiName
    targetFolder := path.Join(outputFolder, t.FolderName(), "services", productName)
    targetFilePath := path.Join(targetFolder, fmt.Sprintf("data_source_%s.go", t.ResourceGoFilename(object)))
    templateData.GenerateDataSourceFile(targetFilePath, object)
}
```

### Condition Check

**File:** `mmv1/api/resource.go`

```go
func (r *Resource) ShouldGenerateSingularDataSource() bool {
    if r.Datasource == nil {
        return false
    }
    return r.Datasource.Generate
}
```

## Template Structure

**File:** `mmv1/templates/terraform/datasource.go.tmpl`

The template generates a data source that:

1. **Reuses the resource schema** via `DatasourceSchemaFromResourceSchema()`
2. **Adds required fields** extracted from `IdFormat`
3. **Adds optional fields** (project, region, zone)
4. **Delegates to the resource Read function**

### Generated Code Structure

```go
func init() {
    registry.Schema{
        Name:        "{{ $.TerraformName }}",
        ProductName: "{{ lower $.ProductMetadata.Name }}",
        Type:        registry.SchemaTypeDataSource,
        Schema:      DataSource{{ .ResourceName }}(),
    }.Register()
}

func DataSource{{ .ResourceName }}() *schema.Resource {
    // Get resource schema and convert to data source schema
    rs := Resource{{ .ResourceName }}().Schema
    dsSchema := tpgresource.DatasourceSchemaFromResourceSchema(rs)

    // Add required fields from IdFormat
    tpgresource.AddRequiredFieldsToSchema(dsSchema, "field1", "field2")

    // Add optional fields (project, region, zone)
    tpgresource.AddOptionalFieldsToSchema(dsSchema, "project")

    return &schema.Resource{
        Read:   dataSource{{ $.ResourceName }}Read,
        Schema: dsSchema,
    }
}

func dataSource{{ $.ResourceName }}Read(d *schema.ResourceData, meta interface{}) error {
    config := meta.(*transport_tpg.Config)

    // Build ID from required fields
    id, err := tpgresource.ReplaceVars(d, config, "{{ $.IdFormat }}")
    if err != nil {
        return err
    }
    d.SetId(id)

    // Delegate to resource Read
    return resource{{ $.ResourceName }}Read(d, meta)
}
```

## Field Extraction from IdFormat

The template automatically extracts fields from the resource's `IdFormat`:

**File:** `mmv1/api/resource.go`

```go
func (r Resource) DatasourceRequiredFields() []string {
    // Extracts {{field}} from IdFormat
    // EXCLUDES: region, project, zone (these become optional)
}

func (r Resource) DatasourceOptionalFields() []string {
    // Only returns: region, project, zone (as applicable)
}
```

**Example:**
- `id_format: 'projects/{{project}}/locations/{{location}}/resources/{{name}}'`
- Required: `location`, `name`
- Optional: `project`

## Registry System

**File:** `mmv1/third_party/terraform/registry/registry.go`

Data sources are registered using the `registry.Schema` system:

```go
type SchemaType int

const (
    SchemaTypeResource SchemaType = iota
    SchemaTypeIAMResource
    SchemaTypeDataSource        // Data sources
    SchemaTypeIAMDataSource
)

type Schema struct {
    Name        string
    ProductName string
    Type        SchemaType
    Schema      *schema.Resource
}
```

The provider retrieves registered data sources via:

```go
func DataSource(name string) *schema.Resource {
    // Returns the registered data source schema
}
```

## Output Files

For a resource named `MyResource` in product `myproduct`:

| File | Path |
|------|------|
| Data source code | `google/services/myproduct/data_source_my_resource.go` |
| Data source test | `google/services/myproduct/data_source_my_resource_test.go` |

## Key Dependencies

The generated data source depends on:

1. **The underlying resource** - Must exist and have a Read function
2. **tpgresource.DatasourceSchemaFromResourceSchema** - Converts resource schema
3. **tpgresource.AddRequiredFieldsToSchema** - Adds required field markers
4. **tpgresource.AddOptionalFieldsToSchema** - Adds optional field markers
5. **Registry system** - For provider registration

## Implications for Ephemeral Resources

1. **Cannot directly reuse SDK v2 code** - Ephemeral resources use Plugin Framework
2. **Schema conversion needed** - Must convert SDK v2 schema to Plugin Framework schema
3. **Different registration** - Plugin Framework uses direct method returns, not registry
4. **Sensitive marking** - Ephemeral resources need explicit `Sensitive: true` on fields
