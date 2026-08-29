# ADR-003: Multi-Tenancy

Status: Accepted
Date: 2026-08-29

## Context
SaaS requires strong tenant isolation while on-premise must remain possible.

## Decision
Use tenant-aware application and data boundaries. Every tenant-owned record carries tenant context.

## Baseline
- authenticated principal resolves to tenant membership
- queries are tenant-scoped
- authorization includes tenant boundary
- background jobs carry tenant context
- audit and events include tenant ID

## Alternatives
- database per tenant
- schema per tenant
- shared database with tenant ID
- hybrid

## Decision detail
Start with shared PostgreSQL database and explicit tenant IDs, while keeping interfaces flexible enough for future isolation tiers.

## Security
Tenant isolation is an invariant and must be covered by negative authorization tests.
