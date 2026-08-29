# CompanyOS Engineering Specification v1.0

Status: Complete — All Gates Approved
Date: 2026-08-29
Scope: Core Platform, CRM, Sales, HR

## 1. Purpose

This package implements the first three engineering artifacts that follow the CompanyOS Product Foundation v1.0:

1. RBAC and Resource Permission Matrix
2. Complete ERD
3. PostgreSQL Schema

Each artifact was delivered in its own approval gate and the next artifact was not drafted until the previous one received explicit `APPROVED` confirmation. All traceability from authorization to logical data to physical schema, and from each accepted invariant to schema controls, has been verified.

## 2. Normative References

- `01-product/companyos-product-blueprint-v1.0.md`
- `02-architecture/architecture-overview-v1.0.md`
- Accepted ADRs in `02-architecture/decisions/` (ADR-001 through ADR-019)
- `02-architecture/domain/core-domain-model-v1.0.md`
- `02-architecture/domain/domain-boundaries-v1.0.md`
- `02-architecture/domain/domain-invariants-v1.0.md`
- `03-governance/architecture-governance-v1.0.md`
- `03-governance/decision-register-v1.0.md`

## 3. Artifacts

| Path | Purpose |
|---|---|
| `04-engineering/security/rbac-resource-permission-matrix-v1.0.md` | Normative authorization model with vocabulary, evaluation contract, scope rules, role/assignment lifecycle, plugin/background policies, and 78-resource permission matrix. |
| `04-engineering/data/complete-erd-v1.0.md` | Logical data model with 15 Mermaid diagrams, 135-entity register, cardinality/optionality, constraint catalog, resource-to-entity and invariant traceability. |
| `04-engineering/data/postgresql-schema-v1.0.sql` | Executable PostgreSQL 16 baseline implementing 135 tables, composite tenant foreign keys, forced RLS, append-only and published-version triggers, optimistic concurrency, capability-version validation, supporting indexes, and role/grants. |
| `04-engineering/data/postgresql-schema-notes-v1.0.md` | Conventions, migration order, transaction contract, role boundaries, retention, ERD-to-schema and RBAC-to-schema traceability, verification record, and known boundaries. |

## 4. Approval Ledger

| Gate | Artifact | Status | Approval date | Approver | Approval phrase |
|---|---|---|---|---|---|
| Gate 1 | RBAC and Resource Permission Matrix v1.0 | Approved | 2026-08-29 | Product owner | `APPROVED - RBAC v1.0` |
| Gate 2 | Complete ERD v1.0 | Approved | 2026-08-29 | Product owner | `APPROVED — Complete ERD v1.0` |
| Gate 3 | PostgreSQL Schema v1.0 + Notes | Approved | 2026-08-29 | Product owner | `APPROVED — PostgreSQL Schema v1.0` |

## 5. Final Consistency Review

- 78 approved RBAC resources are seeded in `core.resource_type` and traced from matrix to ERD register to physical tables.
- 407 exact permission keys are seeded in `authz.permission`. Zero wildcard permissions exist by constraint and seed.
- 135 ERD entity-register entries map one-to-one to 135 PostgreSQL tables. The only deliberate rename is logical `user` to physical `app_user`; logical/RBAC names remain `identity.user` and that alias is documented in the ERD traceability table and Schema Notes.
- All 34 domain invariants (INV-001 through INV-034) are addressed in the RBAC controls, ERD constraints, or Schema Notes.
- Zero cross-plugin private-table foreign keys. Cross-plugin business links use `core.resource` only.
- Zero foreign keys without a supporting prefix index in the verified schema.
- 123 forced Row-Level Security policies cover every tenant-owned table and the platform Tenant table.
- Append-only/append-mostly facts and published-definition children are protected by both runtime privilege controls and database triggers.
- Clean apply of the DDL plus behavioral security tests passed on a clean PostgreSQL 16 cluster. The temporary cluster was stopped and removed after verification.

## 6. Cross-Deliverable Highlights

- `authz.role_assignment` is physically implemented without a free-form `scope_anchor_id`; typed nullable FKs plus check constraints enforce scope type, single anchor, and same-tenant invariant.
- `core.resource` is the registry anchor for comments, attachments, activities, workflow bindings, audit subjects, and cross-module relationships.
- `crm.lead_conversion`, `sales.discount_request`, and `hr.compensation` are isolated aggregates with restricted access and dedicated audit obligations.
- Capability grants are rejected when the capability is not declared by the installed Plugin Version.
- `organization.organization.membership_placement` partial-unique index prevents multiple active primary placements per Membership.
- One published version per Definition is enforced by partial unique indexes on `workflow.workflow_version`, `automation.automation_version`, and `sales.pricing_rule_version`.
- One initial state per published workflow version is enforced by a partial unique index.
- Append-only or immutable facts are guarded by mutation-prevention triggers and runtime grants.

## 7. Out of Scope (Remains Future Engineering Packages)

The following items from the Product Foundation's next-engineering-package list remain separate packages and were not drafted as part of this v1.0:

- API specification
- WebSocket event specification
- Workflow state machine detail
- Activity/event catalog
- Frontend information architecture
- UI design system
- Docker/development architecture
- Windows/on-premise deployment guide
- Backup and recovery plan
- CI/CD implementation specification
- Security threat model and test plan

Any change to the accepted ADRs requires a new ADR per `03-governance/architecture-governance-v1.0.md`. The PostgreSQL Schema is an initial baseline migration; post-v1.0 structural evolution must use forward migrations with immutable checksums.

## 8. Verification Record Summary

- PostgreSQL 16 clean apply: passed.
- Tables: 135.
- Table comments: 135.
- Tenant-isolated tables/policies: 123.
- Seeded Resource Types: 78.
- Seeded exact Permissions: 407.
- Wildcard permissions: 0.
- Unindexed foreign keys: 0.
- Direct cross-plugin private-table foreign keys: 0.
- Behavioral tests: tenant cross-tenant FK rejected, forced RLS filters and blocks cross-tenant inserts, application role cannot read Credentials, published Workflow State mutation denied, draft Field Option becomes immutable after Object Type Version publish, Capability Grant from a different Plugin Version denied, mutable aggregate version increments on update, transaction-local Tenant context does not persist outside its transaction.
- Repeatability: the database was dropped, recreated, and the full DDL reapplied multiple times during validation. Final clean apply and behavior suite passed on 2026-08-29.
- Temporary cluster and database were removed; no test resources remain on the host.

## 9. Conclusion

The three engineering artifacts required to extend the CompanyOS Product Foundation v1.0 are complete, mutually consistent, and approved. Future engineering packages listed in `01-product/companyos-product-blueprint-v1.0.md` may proceed under the governance and traceability contract defined here.
