# CompanyOS Domain Invariants v1.0

## Security invariants

INV-001: Every tenant-owned resource belongs to exactly one tenant.

INV-002: A principal cannot access a resource outside an authorized tenant.

INV-003: Server-side authorization is mandatory.

INV-004: Plugin execution cannot bypass Core authorization.

INV-005: Secrets, passwords, tokens, and private keys are never written to ordinary logs.

INV-006: Audit history cannot be modified through ordinary application operations.

INV-007: Search results must obey resource authorization.

INV-008: Background jobs must preserve tenant and authorization-relevant context.

## Data invariants

INV-009: Canonical business records have stable identifiers.

INV-010: Record mutations are validated against the owning object definition.

INV-011: Derived views are not sources of canonical truth.

INV-012: Business modules cannot directly mutate another module's private persistence.

INV-013: Published workflow versions are immutable.

INV-014: A workflow transition must be valid for the current state.

INV-015: A critical state transition is auditable.

## Event invariants

INV-016: Every event has a unique event ID.

INV-017: Every tenant-scoped event contains tenant context.

INV-018: Event schemas are versioned.

INV-019: Asynchronous handlers must be idempotent where duplicate delivery is possible.

INV-020: Event publication must not silently report success when durable publication failed.

## Work invariants

INV-021: Work has an owner or an explicitly unassigned state.

INV-022: Completed work cannot silently return to active state without a valid transition.

INV-023: Approval decisions are attributable to an authorized actor.

## Plugin invariants

INV-024: Plugin manifests declare dependencies.

INV-025: Plugin migrations are versioned.

INV-026: Plugin permissions are explicit.

INV-027: Plugin lifecycle changes are auditable.

## File invariants

INV-028: File access requires authorization.

INV-029: Uploaded files pass configured validation before becoming available.

INV-030: Object storage keys must not expose predictable tenant/business identifiers.

## Operational invariants

INV-031: Production configuration must not be committed with secrets.

INV-032: Critical releases require passing security gates.

INV-033: Backups must be restorable, not merely created.

INV-034: API clients must not be trusted to enforce business rules.
