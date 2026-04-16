# Ephemeral Resources Unified Generator - Discovery Overview

## Executive Summary

This discovery document explores the feasibility and approach for generating Terraform ephemeral resources alongside data sources in Magic Modules. The goal is to create a unified generator that can produce both resource types from a single definition.

## Background

### What are Ephemeral Resources?

Ephemeral resources are a new Terraform construct introduced in **Terraform v1.10** (November 2024). They are designed to handle sensitive or short-lived data that should never be persisted to state files.

**Key characteristics:**
- Data is **never stored** in state or plan files
- Executed on **every plan and apply** (generating fresh values)
- Use a different lifecycle: `Open`, `Renew` (optional), `Close` (optional)
- Reference prefix: `ephemeral.<resource_name>`
- Only referenceable in temporary contexts (provider configs, ephemeral locals, other ephemeral resources)

### Why Unified Generation?

Ephemeral resources and data sources are conceptually similar:
1. Both are **read-only** operations
2. Both retrieve data from external sources
3. Both have similar schema structures
4. The main difference is **state persistence** and **security sensitivity**

A unified generator would:
- Reduce code duplication
- Ensure consistency between data sources and ephemeral resources
- Simplify maintenance
- Enable automatic ephemeral resource generation for existing data sources

## Current State in terraform-provider-google

### Existing Ephemeral Resources (Handwritten)

Located in `mmv1/third_party/terraform/services/`:

| Resource | File | Purpose |
|----------|------|---------|
| `google_client_config` | `resourcemanager/ephemeral_google_client_config.go` | Provider client configuration |
| `google_service_account_access_token` | `resourcemanager/ephemeral_google_service_account_access_token.go` | Temporary access tokens |
| `google_service_account_id_token` | `resourcemanager/ephemeral_google_service_account_id_token.go` | Temporary identity tokens |
| `google_service_account_jwt` | `resourcemanager/ephemeral_google_service_account_jwt.go` | Service account JWTs |
| `google_service_account_key` | `resourcemanager/ephemeral_google_service_account_key.go` | Temporary service account keys |
| `google_secret_manager_secret_version` | `secretmanager/ephemeral_google_secret_manager_secret_version.go` | Secret version data |

### Generation Status

| Type | Generated | Framework |
|------|-----------|-----------|
| Resources | Yes (mmv1) | SDK v2 |
| Data Sources | Yes (mmv1) | SDK v2 |
| Ephemeral Resources | **No (handwritten only)** | Plugin Framework |

## Key Findings

1. **Different Frameworks**: Data sources use SDK v2, ephemeral resources use Plugin Framework
2. **Different Registration**: SDK v2 uses registry system, Plugin Framework uses direct provider registration
3. **Schema Similarity**: Ephemeral resources share ~90% of schema with corresponding data sources
4. **Sensitive Marking**: Ephemeral resources explicitly mark sensitive attributes

## Scope of This Discovery

1. Understand current data source generation pipeline
2. Understand ephemeral resource implementation patterns
3. Identify integration points for unified generation
4. Propose architecture options
5. Identify technical challenges and solutions

## Documents in This Discovery

- `01-overview.md` - This overview document
- `02-current-data-source-generation.md` - How data sources are currently generated
- `03-current-ephemeral-resources.md` - Analysis of existing handwritten ephemeral resources
- `04-architecture-options.md` - Proposed approaches for unified generation
- `05-technical-considerations.md` - Technical challenges and solutions
- `06-implementation-plan.md` - Recommended implementation approach
