# CompanyOS PostgreSQL Schema Notes v1.0

Status: Approved — Gate 3
Date: 2026-08-29
Approved: 2026-08-29
Schema artifact: `postgresql-schema-v1.0.sql`
Normative inputs:
- RBAC and Resource Permission Matrix v1.0 — Approved Gate 1
- Complete ERD v1.0 — Approved Gate 2

## 1. Scope

The DDL implements all 135 logical entities in the approved Complete ERD for Core, CRM, Sales, and HR. It includes tenant isolation, exact permission seeds, module ownership boundaries, same-tenant foreign keys, append-only and published-version protections, optimistic concurrency, operational indexes, runtime database roles, and migration ordering.

The file is an initial baseline migration for an empty PostgreSQL database. It is not an idempotent full-schema rerun. Seed statements are conflict-safe; structural evolution after v1.0 must use forward migrations.

## 2. Supported PostgreSQL Contract

- Minimum PostgreSQL version: 16.
- Required extension: `pgcrypto`, used only for `gen_random_uuid()`.
- Encoding: UTF-8.
- Time zone: application and database sessions use UTC; all instants use `timestamptz`.
- Collation: deployment-defined. Business keys requiring case-insensitive behavior are normalized at the application boundary until a dedicated collation policy is approved.
- Transactionality: the complete baseline runs inside one transaction.
- Object binaries remain outside PostgreSQL in S3-compatible storage; PostgreSQL stores metadata and checksums.

`pgcrypto` is bundled with standard PostgreSQL/contrib distributions and remains compatible with SaaS, private-cloud, and on-premise deployment targets.

## 3. Physical Namespace Map

| Schema | Logical owner | Notes |
|---|---|---|
| `core` | Foundation | Tenant, Resource Type, Resource registry, shared types/functions. |
| `identity_core` | Identity | Global User/authentication and tenant-aware identity-provider/service-principal data. Named to avoid ambiguous `identity` terminology in tooling. |
| `organization` | Organization | Company hierarchy, Membership, placements, teams, hierarchy projection. |
| `authz` | Authorization | Physical name avoids PostgreSQL `AUTHORIZATION` keyword; logical namespace and permission keys remain `authorization.*`. |
| `core_data` | Core Data + Search | Custom objects, fields, records, relationships, saved queries. |
| `work` | Work | Task, Project, Milestone, Request, assignments and history. |
| `workflow` | Workflow | Definitions, immutable versions, runtime instances, transitions and approvals. |
| `content` | Content | File metadata, scanning, Document/version, Folder, Attachment and upload session. |
| `communication` | Communication | Comment/revision, Mention, Notification/delivery and Subscription. |
| `automation` | Automation | Versioned definitions and execution results. |
| `observability` | Activity + Audit | Activity, append-only audit and security events. |
| `eventing` | Event + Reliability | Domain events, outbox, subscription, delivery and idempotency. |
| `plugin` | Plugin | Catalog, versions, dependencies, migrations, installations, capabilities and configuration. |
| `configuration` | Configuration | Tenant and feature settings. |
| `integration` | Integration | Connections, webhooks and delivery. |
| `crm` | CRM plugin | Nine CRM-owned entities. |
| `sales` | Sales plugin | Nine Sales-owned entities. |
| `hr` | HR plugin | Fourteen HR-owned entities. |

Core has no foreign key to a plugin-private table. No plugin has a foreign key to another plugin-private table. Cross-plugin references use `core.resource`.

## 4. Creation and Migration Order

The baseline uses this dependency order:

1. extension, schemas, enums and shared functions;
2. Tenant, Resource Type and global User identity;
3. organization hierarchy and Membership;
4. tenant-aware identity providers and service principals;
5. Resource registry;
6. Authorization;
7. Core Data and Search;
8. Work;
9. Workflow and Approval;
10. Content and Communication;
11. Automation, Eventing and Observability;
12. Plugin, Configuration and Integration;
13. CRM, Sales and HR;
14. cross-table version constraints and operational indexes;
15. immutability/version triggers, FK indexes and comments;
16. RLS policies and database roles/grants;
17. registered resource and exact permission seeds.

Future plugin migrations execute only after Core baseline migrations and their declared dependencies. `plugin.plugin_migration` records immutable keys, sequence, and checksums; the application migration runner remains responsible for applying files and recording execution state in its migration framework.

## 5. Identifier, Type, and Naming Rules

- Primary keys are UUIDs generated with `gen_random_uuid()` unless deterministic external provisioning supplies one.
- Tenant-owned tables expose `UNIQUE (tenant_id, id)` to support same-tenant composite foreign keys.
- Names use lowercase `snake_case`; physical schemas identify module ownership.
- Human-readable business keys are text alternate keys scoped to tenant and owning aggregate.
- Monetary values use `numeric(19,4)`; quantities and ratios use explicit higher scales where needed.
- Currency uses three-character ISO-style codes. ISO membership validation belongs in a reference catalog/API boundary when currency support is implemented.
- Instants use `timestamptz`; business dates use `date`.
- Mutable aggregate roots use `version bigint` for optimistic concurrency and `updated_at` maintained by trigger.
- JSONB is limited to controlled configuration, validated expressions, safe payloads, and genuinely extensible values. Canonical relational links remain foreign keys.
- Raw credentials, tokens and integration secrets are not stored in ordinary configuration JSON. Authentication secrets are one-way hashes; retrievable secrets are external secret references.

## 6. Tenant Integrity

### 6.1 Structural Integrity

All tenant-owned relationships use `(tenant_id, referenced_id)` composite foreign keys wherever both ends are relational entities. This blocks cross-tenant association even under application defects or privileged migration code.

The approved polymorphic scope design is physically implemented without a free-form `scope_anchor_id`. `authz.role_assignment` has typed nullable FKs for Organization, Business Unit, Department, Team, Location, owner Membership, and Resource. Check constraints enforce:

- no anchor for `tenant` scope;
- exactly one anchor for every narrower scope;
- anchor column matches `scope_type`;
- every anchor is same-tenant.

Cross-domain generic references target `core.resource`, which has a unique `(tenant_id, resource_type_id, concrete_id)` registration key and authorization projection columns.

### 6.2 Row-Level Security

RLS is implemented as defense in depth:

- every table with `tenant_id` has `ENABLE ROW LEVEL SECURITY` and `FORCE ROW LEVEL SECURITY`;
- `core.tenant` has a special policy matching its `id`;
- 123 tenant isolation policies exist in the verified schema;
- policy expression compares tenant identity with `core.current_tenant_id()`;
- missing or malformed tenant context does not produce an allow;
- runtime roles are `NOBYPASSRLS`.

RLS is not the complete RBAC engine. It enforces tenant boundary only. Resource permission, scope hierarchy, ownership, field classification, workflow state, separation of duties, plugin capability, and export rules remain application-service checks defined by the approved RBAC specification.

### 6.3 Transaction Contract

Every tenant-owned application transaction MUST use this sequence on one checked-out connection:

```sql
BEGIN;
SET LOCAL ROLE companyos_app;
SELECT set_config('companyos.tenant_id', :tenant_id, true);
-- authorized statements
COMMIT;
```

Identity, worker, and auditor transactions use their assigned database role and set the same transaction-local tenant context when accessing tenant-owned rows.

Requirements:

1. resolve and validate the authenticated principal and active Membership before beginning business work;
2. set tenant context with `is_local = true` inside the transaction;
3. never use session-persistent tenant context on pooled connections;
4. execute authorization and mutation in the same transaction when stale context could alter the result;
5. rollback on missing context, role mismatch, authorization uncertainty, or partial failure;
6. migration connections use a separate controlled role and MUST NOT serve application requests.

The verification suite intentionally confirmed that transaction-local context disappears outside its transaction.

## 7. Database Role Boundary

| Role | Purpose | RLS | Access baseline |
|---|---|---|---|
| `companyos_migrator` | Controlled structural migrations and bootstrap | `BYPASSRLS` | No login; deployment-specific login inherits it. Never used by runtime. |
| `companyos_identity` | Identity/authentication service | `NOBYPASSRLS` | Identity tables plus tenant/Membership reads; separate from ordinary business app. |
| `companyos_app` | Interactive application use cases | `NOBYPASSRLS` | Business tables; no direct Credential table privileges. |
| `companyos_worker` | Event/outbox/background delivery | `NOBYPASSRLS` | Event status work and append-only observability inserts. |
| `companyos_auditor` | Restricted audit/security review | `NOBYPASSRLS` | Read-only Audit Event and Security Event under tenant policy. |

Roles are `NOLOGIN` group roles. Deployment creates login roles and grants only the required group role. Public schema privileges and application-schema privileges are revoked from `PUBLIC`.

Database grants are coarse module/boundary controls. They do not replace application RBAC.

## 8. Authorization Persistence

- `authz.permission` stores only exact registered keys; `CHECK (key !~ '\\*')` rejects wildcards.
- The seed is derived from the approved matrix and contains 407 exact permissions across 78 registered resource types.
- Permission key is constrained to `owner_module.resource_key.action`.
- Role inheritance and runtime deny grants have no schema representation, matching approved v1.0 decisions.
- Role Permission is a tenant-bound many-to-many association.
- Role Assignment and Service Role Assignment are first-class scoped grants.
- Active human assignment uniqueness prevents duplicate effective grants with the same role/scope anchor.
- Role Assignment Event is append-only.
- Role/permission grant-boundary checks, self-escalation prevention, expiry evaluation, and cache invalidation are application authorization responsibilities.

Reserved baseline roles are not globally seeded because each tenant requires scoped, auditable bootstrap. Tenant provisioning creates reserved role rows and their approved grants transactionally.

## 9. Custom Record Storage

Custom object definitions are versioned:

- `core_data.object_type` is the stable identity;
- `object_type_version` is draft/published/retired;
- Field Definition and Field Option belong to one version;
- children of published versions are protected from update/delete;
- Record identifies the canonical instance and registered Resource;
- Record Value stores exactly one typed value column per record/field.

The physical typed columns are text, integer, decimal, boolean, date, timestamp, UUID, and JSONB. The application validates the populated column against `field_definition.data_type` inside the mutation transaction. Gate 3 deliberately avoids a giant untyped JSON document as canonical record storage.

Search starts PostgreSQL-first. Resource scope, object type, status, ownership projection and key business lookup indexes are present; generated full-text vectors and field-specific indexes are added by later migrations only for approved object definitions and measured query needs.

## 10. Resource Registry Contract

Every securable aggregate root is inserted with a matching `core.resource` row in the same transaction. The registry stores:

- registered Resource Type;
- concrete aggregate ID;
- Tenant;
- authorization-relevant Organization/Business Unit/Department/Team/Location/owner projection;
- lifecycle and projection version.

The owning module remains responsible for validating that Resource Type matches the concrete table. This cannot be expressed as a static foreign key across multiple possible tables. Application integration tests MUST prove registration is atomic and type-correct for each aggregate.

A missing, stale, or mismatched authorization projection fails access closed. Projection updates and aggregate scope movement occur in one transaction and increment projection version.

## 11. Versioning and Immutability

### 11.1 Optimistic Concurrency

Tables containing both `updated_at` and `version` receive a generated `set_updated_at` trigger. Updates atomically increment version. Application updates use:

```sql
UPDATE schema.table
SET ...
WHERE tenant_id = :tenant_id
  AND id = :id
  AND version = :expected_version;
```

A zero-row result is a conflict, not a silent success.

### 11.2 Published Definitions

Published Workflow, Object Type, Automation, Pricing Rule, and Onboarding Plan children cannot be changed through ordinary update/delete paths. Runtime Workflow constraints additionally enforce that:

- transition source and target states belong to the transition's Workflow Version;
- current Workflow Instance state belongs to its Workflow Version;
- transition condition/action references share the transition version.

Publishing, retirement and selected-current-version changes are controlled application transitions and audited.

### 11.3 Append-Only Facts

Mutation-prevention triggers protect:

- Role Assignment Event;
- Work Status History;
- Transition Execution and Approval Decision;
- File Scan and Document Version;
- Comment Revision;
- Activity, Audit Event and Security Event;
- Domain Event;
- Plugin Lifecycle Event;
- CRM Lead Conversion;
- Sales Discount Decision;
- submitted HR Interview Feedback.

Runtime grants additionally revoke update/delete/truncate from key audit/event tables. Outbox and delivery rows allow controlled operational status updates, while linked Domain Event payloads remain immutable.

## 12. Index Strategy

The schema creates:

- primary and alternate-key indexes;
- partial uniqueness for one active assignment/current published version/initial state;
- resource authorization scope index;
- work inbox and approval assignee indexes;
- notification inbox index;
- outbox/event/webhook pending-queue indexes;
- audit/security/event time indexes;
- CRM, Sales and HR operational lookup indexes;
- candidate retention and effective Employment indexes;
- generated prefix-supporting indexes for every foreign key not already covered.

Verified result: zero foreign keys lack a supporting prefix index.

Indexes intentionally avoid premature analytics denormalization. Dashboard/search projections are introduced only with explicit rebuild and authorization contracts.

## 13. Hierarchy and Effective-Date Controls

Self-parent checks prevent immediate self-cycles. Organization and Folder closure tables support descendant authorization and hierarchy queries. Application hierarchy mutation must:

1. lock affected hierarchy roots;
2. reject any move that makes a node its own ancestor;
3. update canonical parent and closure rows atomically;
4. update affected Resource authorization projections;
5. invalidate authorization caches;
6. emit audit/domain events.

The baseline validates date order and current-primary uniqueness. Non-overlapping arbitrary effective intervals require PostgreSQL exclusion constraints after the exact overlap semantics and zero-downtime extension strategy are approved. Until then, application transactions lock the owning entity and reject prohibited overlaps.

## 14. Plugin and Cross-Module Controls

- Plugin Version, dependencies, capability definitions and migrations are immutable catalog concepts.
- Plugin Installation and grants are tenant-owned and RLS protected.
- A trigger rejects a Capability Grant not declared by the installed Plugin Version.
- Plugin configuration stores either non-secret JSON or a secret reference.
- Cross-plugin business references use `core.resource`.
- Physical database roles do not grant plugins direct table access; plugins execute through owning application contracts.
- Plugin migrations run with controlled migrator privileges and are reviewed for schema ownership and forbidden cross-module dependencies.

## 15. Retention and Privacy

| Data class | Physical approach |
|---|---|
| Audit/security/domain facts | Append-only rows; payload minimization; restrictive parent FKs; policy-governed retention. |
| Credentials/session secrets | Hash/reference only; revocation first; timed purge without deleting attribution events. |
| File binaries | Object storage lifecycle; PostgreSQL metadata/checksum/state retained per business policy. |
| HR Compensation | Separate restricted table; no ordinary app-role implication of read authorization; reads require application audit. |
| Candidate/contact personal data | Archival/erasure workflow with retention deadline; preserve non-sensitive accountability references. |
| Notifications/deliveries/idempotency | Bounded operational retention and safe deletion jobs. |
| Published definitions and decisions | Immutable for referential and accountability integrity. |

The schema does not encode jurisdiction-specific durations. Backup retention must be aligned with deletion policy, and restore procedures must prevent erased sensitive data from silently re-entering active systems.

## 16. Migration and Recovery Rules

- Do not rerun the structural baseline against a populated database.
- Every post-v1.0 change is a numbered forward migration with immutable checksum.
- Prefer additive expand/migrate/contract changes for live environments.
- Destructive migrations require backup/restore evidence, data migration verification, architecture/security review, and an explicit rollout plan.
- `CREATE INDEX CONCURRENTLY` cannot run inside a transaction and therefore belongs in a dedicated operational migration when needed at scale.
- Enum expansion is forward-only in practice; frequently changing business statuses should migrate to reference tables before volatility appears.
- Rollback means application rollback plus a tested forward corrective migration unless a structural reversal is proven data-safe.
- Plugin uninstall archives installation/configuration state and follows owner retention policy; it does not blindly drop referenced data.

## 17. ERD-to-Schema Traceability

| ERD group | Physical schemas/tables | Key controls |
|---|---|---|
| Foundation and Identity | `core.*`, `identity_core.*` | Global User separation, Tenant RLS, secret hashes/references, identity role. |
| Organization | `organization.*` | Same-tenant hierarchy and Membership FKs, effective placements, closure projection. |
| Authorization | `authz.*` | Exact keys, no wildcard, flat Role, typed scoped assignment. |
| Core Data/Search | `core_data.*` | Versioned definitions, typed Record Values, Resource relationships. |
| Work | `work.*` | Explicit assignee XOR, dependencies, append-only status history. |
| Workflow/Approval | `workflow.*` | Same-version constraints, immutable published graph, attributable decisions/delegation. |
| Content | `content.*` | Binary metadata only, scan history, source XOR Attachment, Document versions. |
| Communication | `communication.*` | Same-tenant parent/member links, revisions, recipient-owned notifications. |
| Automation | `automation.*` | Immutable published definitions, execution/idempotency links. |
| Observability/Event | `observability.*`, `eventing.*` | Append-only facts, outbox, queue indexes, correlation and idempotency. |
| Plugin/Configuration/Integration | `plugin.*`, `configuration.*`, `integration.*` | Capability-version validation, secret references, audited lifecycle/delivery. |
| CRM | `crm.*` | Nine entities; Resource-based cross-module links. |
| Sales | `sales.*` | Nine entities; parent-bound lines, monetary checks, discount XOR/decision. |
| HR | `hr.*` | Fourteen entities; restricted Compensation, recruitment and onboarding boundaries. |

All 135 Entity Register entries have one physical table. The single deliberate rename is the global `identity.user` aggregate, stored as `identity_core.app_user` because `user` is syntactically ambiguous and often reserved in SQL tooling; logical name and RBAC resource remain `identity.user`. Other logical-to-physical names are direct within their owner schema.

## 18. RBAC-to-Schema Traceability

- 78 approved RBAC resources are seeded in `core.resource_type`.
- 407 approved action combinations are seeded in `authz.permission`.
- Role/permission assignments map to `authz.role`, `role_permission`, `role_assignment`, and `service_role_assignment`.
- Organization scopes map to typed FKs and `organization.org_hierarchy_closure`.
- Owner scope maps to Membership and Resource owner projection.
- Record scope maps to `core.resource`.
- Plugin capability checks map to Plugin Installation, Capability Definition and Capability Grant.
- Sensitive HR resources map to separate physical aggregates rather than fields on Employee.
- Audit-required actions persist to append-only observability entities through application transaction/outbox contracts.

The database stores authorization facts and enforces Tenant integrity. The approved evaluation order and contextual decisions remain normative in the RBAC document.

## 19. Verification Record

Verification ran against an isolated clean local PostgreSQL cluster using a short Unix socket and a dedicated temporary database. The cluster was stopped after each run.

### 19.1 Clean Apply

- complete DDL applied inside one transaction;
- resulting application tables: 135;
- table comments: 135;
- Tenant RLS policies: 123;
- seeded Resource Types: 78;
- seeded exact Permissions: 407;
- wildcard Permissions: 0;
- foreign keys without supporting prefix index: 0.

### 19.2 Behavioral Tests

Passed:

- cross-tenant composite FK insertion denied;
- RLS returns only the transaction Tenant;
- RLS denies cross-tenant insertion by `companyos_app`;
- `core.tenant` itself is Tenant-filtered;
- application role cannot read Credential rows;
- published Workflow State mutation denied;
- draft Field Option remains editable and becomes immutable after its Object Type Version is published;
- Capability Grant from a different Plugin Version is denied;
- mutable aggregate version increments on update;
- seed statements reapply without duplicate rows;
- transaction-local Tenant context does not persist outside its transaction.

### 19.3 Repeatability

The database was dropped, recreated, and the full DDL reapplied multiple times during validation. Final clean apply and behavior suite passed on 2026-08-29.

## 20. Known Boundaries

These are implementation obligations, not unresolved logical-model changes:

- application services must atomically maintain Resource registration/projection;
- application authorization must enforce scopes beyond Tenant, field classification, SoD, workflow conditions and grant boundaries;
- hierarchy mutations and arbitrary effective-date overlap require transaction-level services;
- retention durations, legal erasure and backup expiry need the later Backup/Recovery and Security packages;
- full-text/generated custom-field indexes are introduced from measured query requirements;
- login credentials and ownership of physical schemas are deployment/migration-runner configuration;
- outbox publisher uses controlled status-column updates and immutable Domain Event payloads;
- table partitioning is deferred until volume evidence exists.

## 21. Gate 3 Review Summary

- PostgreSQL baseline: 16+
- Physical schemas: 18
- Tables: 135
- Tenant-isolated tables/policies: 123
- Registered RBAC resources: 78
- Seeded exact permissions: 407
- Foreign keys without supporting index: 0
- Direct cross-plugin private-table dependencies: 0
- Clean apply: Passed
- Behavioral security tests: Passed

## 22. Approval Gate

| Gate | Artifact | Status | Approval date | Approver | Notes |
|---|---|---|---|---|---|
| Gate 1 | RBAC + Resource Permission Matrix v1.0 | Approved | 2026-08-29 | Product owner | Normative authorization input. |
| Gate 2 | Complete ERD v1.0 | Approved | 2026-08-29 | Product owner | Normative logical data input. |
| Gate 3 | PostgreSQL Schema v1.0 + Notes | Approved | 2026-08-29 | Product owner | Approved in conversation with `APPROVED — PostgreSQL Schema v1.0`. |

Approval phrase: `APPROVED — PostgreSQL Schema v1.0`
