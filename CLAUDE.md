# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Magic Modules is a code generator that produces the Terraform providers for Google Cloud: `terraform-provider-google` (GA) and `terraform-provider-google-beta`. Contributors make changes here and the `modular-magician` robot generates the downstream provider code.

## Build Commands

```bash
# Check dependencies (go 1.26+, git, terraform, make)
make doctor

# Generate provider code (requires OUTPUT_PATH and VERSION env vars)
OUTPUT_PATH=/path/to/terraform-provider-google VERSION=ga make build

# Generate for a specific product only
OUTPUT_PATH=/path/to/provider VERSION=ga PRODUCT=compute make build

# Generate for a specific resource within a product
OUTPUT_PATH=/path/to/provider VERSION=ga PRODUCT=compute RESOURCE=Address make build

# Run mmv1 tests
make test
```

## Architecture

### Code Generators

**mmv1/** - The primary generator (written in Go)
- `main.go` - Entry point, parses flags and orchestrates generation
- `products/` - YAML definitions for each GCP product/service
  - `product.yaml` - Product metadata (name, API versions, base URLs)
  - `ResourceName.yaml` - Resource definitions with properties, examples, custom code references
- `api/` - Go types for parsing YAML (Resource, Product, Type, etc.)
- `provider/` - Provider implementations (terraform.go, terraform_tgc.go, etc.)
- `templates/terraform/` - Go templates for generating Terraform resource code
- `third_party/terraform/` - Handwritten Terraform provider code that gets copied directly
  - `services/` - Handwritten resources organized by service
- `loader/` - Loads and validates product/resource YAML files

**tpgtools/** - DCL-based resource generator (secondary)
- Uses OpenAPI specs from `api/` directory
- Overrides in `overrides/` directory
- For resources implemented via the Declarative Client Library

### CI System

**.ci/magician/** - Go-based CI automation tool
- Handles PR workflows, downstream generation, VCR testing
- Commands in `cmd/` (generate_comment, generate_downstream, test_terraform_vcr, etc.)

### Downstream Repositories

Generated code goes to:
- `terraform-provider-google` (GA) - tracked by `tpg-sync` branch
- `terraform-provider-google-beta` - tracked by `tpgb-sync` branch
- `terraform-google-conversion` - tracked by `tgc-sync` branch
- `docs-examples` (Open in Cloud Shell) - tracked by `tf-oics-sync` branch

## Resource Definition Structure

Resources are defined in `mmv1/products/<product>/<Resource>.yaml`:

```yaml
name: 'ResourceName'
description: |
  Resource description for documentation
base_url: 'projects/{{project}}/locations/{{region}}/resources'
min_version: 'beta'  # Optional, defaults to ga
immutable: true      # Optional, resource cannot be updated
async:               # For async operations
  type: 'OpAsync'
  operation:
    base_url: '{{op_id}}'
custom_code:         # Reference custom Go template files
  constants: 'templates/terraform/constants/resource.go.tmpl'
  encoder: 'templates/terraform/encoders/resource.go.tmpl'
examples:
  - name: 'example_basic'
    primary_resource_id: 'default'
    vars:
      resource_name: 'my-resource'
properties:
  - name: 'propertyName'
    type: String
    description: 'Property description'
    required: true
```

## Key Files for Development

- `mmv1/api/resource.go` - Resource struct with all YAML field definitions
- `mmv1/api/type.go` - Property type definitions (String, Integer, NestedObject, etc.)
- `mmv1/templates/terraform/resource.go.tmpl` - Main resource generation template
- `mmv1/provider/terraform.go` - Terraform provider generation logic

## Version Handling

- `ga` - General availability, resources go to both GA and beta providers
- `beta` - Beta-only resources (set via `min_version: 'beta'`)
- When generating GA, docs are generated for beta features but not code

## Testing

Tests are generated from `examples` in resource YAML files. The example configs become acceptance tests in the downstream providers.
