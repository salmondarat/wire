# CompanyOS Product Foundation v1.0

Status: Baseline Architecture Package
Date: 2026-08-29
Scope: Product Blueprint, Architecture Decision Records, Core Domain Model, Domain Invariants, Governance Baseline

## Purpose

This package is the foundational specification for CompanyOS, a modular company operating system intended to connect company-wide people, work, data, workflows, documents, events, and management visibility.

The platform is industry-neutral. Business-specific capabilities are delivered through plugins/modules. The initial architecture is a modular monolith with explicit domain boundaries and a future path toward selective service extraction.

## Documents

- `01-product/companyos-product-blueprint-v1.0.md`
- `02-architecture/architecture-overview-v1.0.md`
- `02-architecture/decisions/ADR-001-product-architecture.md` through `ADR-019-deployment-model.md`
- `02-architecture/domain/core-domain-model-v1.0.md`
- `02-architecture/domain/domain-boundaries-v1.0.md`
- `02-architecture/domain/domain-invariants-v1.0.md`
- `03-governance/architecture-governance-v1.0.md`

## Decision Status

All decisions in this package are baseline decisions for implementation planning. Decisions may be superseded only through a new ADR.

## Next engineering package

The next package should define:
1. RBAC + Resource Permission Matrix
2. Complete ERD
3. PostgreSQL schema
4. API specification
5. WebSocket event specification
6. Workflow state machine
7. Activity/event catalog
8. Frontend information architecture
9. UI design system
10. Docker/development architecture
11. Windows/on-premise deployment guide
12. Backup and recovery plan
13. CI/CD implementation specification
14. Security threat model and test plan
