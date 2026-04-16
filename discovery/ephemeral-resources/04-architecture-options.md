# Architecture Options for Unified Generation

## Overview

This document explores different architectural approaches for generating both data sources and ephemeral resources from a unified definition.

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
