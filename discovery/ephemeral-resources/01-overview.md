# Ephemeral Resources Generator - Discovery Overview

## Executive Summary

This discovery document explores the feasibility and approach for generating Terraform ephemeral resources in Magic Modules.

### ⚠️ Key Finding: Direct Wrapper is Blocked

After investigation and team discussion, **direct wrapper pattern (calling data source Read from ephemeral) is not currently feasible** because:

1. Generated data sources delegate to `resourceXxxRead()` which is SDK v2 code
2. Ephemeral resources must use Plugin Framework
3. Can't directly call SDK v2 code from Plugin Framework
4. Would require completing the full SDK v2 → Plugin Framework migration first

### 🔍 New Investigation: Go Reflect Bridge (In Progress)

Exploring whether Go's `reflect` package can provide a **runtime bridge** between SDK v2 and Plugin Framework. See `09-go-reflect-approach.md` for details.

Key insight: SDK v2 has `Resource.Data()` method that creates `*ResourceData` at runtime, enabling:
1. Create `*ResourceData` from data source schema
2. Populate with input values from Plugin Framework
3. Call SDK v2 Read function
4. Extract values and convert back to Plugin Framework types

**Status**: Potentially feasible - needs prototype to validate.

### Fallback Approach: Standalone Ephemeral Generation

If reflect bridge doesn't work, generate **standalone ephemeral resources** with their own API logic. See `08-standalone-ephemeral-generation.md` for details.

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
5. **BLOCKER**: Data sources delegate to resource Read (SDK v2), so can't wrap them from Plugin Framework

## Scope of This Discovery

1. Understand current data source generation pipeline
2. Understand ephemeral resource implementation patterns
3. ~~Identify integration points for unified generation~~ (BLOCKED)
4. Propose architecture options
5. Identify technical challenges and solutions

## Conclusion

| Approach | Status | Notes |
|----------|--------|-------|
| **Direct wrapper (call SDK v2 from PF)** | ❌ BLOCKED | Can't directly call SDK v2 from Plugin Framework |
| **Go reflect bridge** | 🔍 INVESTIGATING | Use reflect to bridge frameworks at runtime |
| **Standalone ephemeral generation** | ✅ VIABLE | Fallback - generate with own API logic |

## Documents in This Discovery

- `01-overview.md` - This overview document
- `02-current-data-source-generation.md` - How data sources are currently generated
- `03-current-ephemeral-resources.md` - Analysis of existing handwritten ephemeral resources
- `04-architecture-options.md` - Proposed approaches (includes blocker explanation)
- `05-technical-considerations.md` - Technical challenges and solutions
- `06-implementation-plan.md` - Implementation approach (needs update)
- `07-key-code-references.md` - Quick reference for key files
- `08-standalone-ephemeral-generation.md` - Standalone generation approach (fallback)
- `09-go-reflect-approach.md` - Go reflect bridge investigation (technical)
- `10-reflect-for-beginners.md` - **NEW: Beginner-friendly reflect explanation**
