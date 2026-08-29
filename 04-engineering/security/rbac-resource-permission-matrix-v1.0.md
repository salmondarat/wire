# CompanyOS RBAC and Resource Permission Matrix v1.0

Status: Approved — Gate 1
Date: 2026-08-29
Approved: 2026-08-29
Scope: Core Platform, CRM, Sales, HR

## 1. Purpose

This specification defines the authorization contract for CompanyOS. It converts ADR-005 into an implementation-grade RBAC model combining roles, atomic resource permissions, organizational scopes, ownership, and contextual policy checks.

This document is normative for the Complete ERD and PostgreSQL Schema. Those artifacts must not be drafted until this document is approved.

## 2. Normative References

- `01-product/companyos-product-blueprint-v1.0.md`
- `02-architecture/architecture-overview-v1.0.md`
- `02-architecture/decisions/ADR-003-multi-tenancy.md`
- `02-architecture/decisions/ADR-004-identity-model.md`
- `02-architecture/decisions/ADR-005-authorization-model.md`
- `02-architecture/decisions/ADR-006-core-domain-boundary.md`
- `02-architecture/decisions/ADR-007-plugin-architecture.md`
- `02-architecture/decisions/ADR-009-workflow-engine.md`
- `02-architecture/decisions/ADR-010-event-architecture.md`
- `02-architecture/decisions/ADR-011-audit-architecture.md`
- `02-architecture/domain/core-domain-model-v1.0.md`
- `02-architecture/domain/domain-boundaries-v1.0.md`
- `02-architecture/domain/domain-invariants-v1.0.md`
- `03-governance/architecture-governance-v1.0.md`

## 3. Normative Language

`MUST`, `MUST NOT`, `SHOULD`, `SHOULD NOT`, and `MAY` are normative requirements.

## 4. Authorization Principles

1. Every decision is enforced server-side and defaults to deny.
2. Authentication alone grants no resource access.
3. A principal MUST have an active tenant membership before tenant-owned access is evaluated.
4. Tenant isolation is evaluated before roles, permissions, scopes, ownership, or workflow state.
5. A permission is an explicit allow for one action on one resource type; CompanyOS has no runtime deny grants.
6. Role assignments are tenant-specific and scope-specific.
7. Multiple valid assignments combine by union, but no union can cross a tenant boundary.
8. Ownership is a scope condition, not an implicit universal privilege.
9. Organizational Position and security Role are independent concepts.
10. UI visibility is never an authorization control.
11. Plugins MUST use Core authorization and cannot grant themselves capabilities.
12. Search, exports, background jobs, automations, and event consumers obey the same resource authorization contract.
13. Security-sensitive and material authorization decisions are auditable.
14. Permission checks use stable resource attributes captured inside the use-case transaction where consistency matters.

## 5. Authorization Vocabulary

| Term | Definition |
|---|---|
| Principal | Authenticated human, service, background worker, or plugin identity requesting an action. |
| User | Global system identity; it is not itself a tenant authorization grant. |
| Membership | Relationship connecting a user to one tenant and its organization context. |
| Role | Named collection of permissions, either reserved by the platform or custom within a tenant. |
| Permission | Atomic action on a resource, named `domain.resource.action`. |
| Role assignment | Binding of one role to one membership at exactly one scope anchor. |
| Scope type | Kind of boundary: `tenant`, `organization`, `business_unit`, `department`, `team`, `location`, `owner`, or `record`. |
| Scope anchor | Identifier that qualifies a scope type; tenant and owner use contextual anchors. |
| Resource | Tenant-owned or platform-owned object being accessed. |
| Resource context | Trusted attributes used to evaluate tenant, scope, owner, state, sensitivity, and module ownership. |
| Capability | Explicit contract authorizing a plugin principal to call a Core or plugin-owned application service. |
| Contextual policy | Non-RBAC condition such as workflow state, assignment, separation of duties, or step-up authentication. |
| Effective access | Union of all active, matching explicit allows after tenant, scope, capability, and contextual checks pass. |

## 6. Principal Classes and Trust Boundaries

| Principal class | Authentication | Membership requirement | Authorization source | Restrictions |
|---|---|---|---|---|
| Human member | Interactive session or supported identity provider | Active membership in target tenant | Scoped role assignments | Subject to session, membership, scope, resource, and contextual checks. |
| Tenant administrator | Human member with reserved `Tenant Administrator` role | Active tenant membership | Reserved role at tenant scope | Cannot bypass platform invariants, read credentials, modify audit history, or self-approve prohibited actions. |
| Service principal | Strong non-human credential | Explicit tenant binding unless platform-operated | Scoped service role | No interactive-only actions; credentials are rotated and never exposed to plugins. |
| Background job principal | Signed/internal job identity plus captured initiating context | Tenant context is mandatory for tenant work | Minimum system capability plus initiating actor context where attribution matters | MUST reject missing, expired, or invalid authorization-relevant context. |
| Plugin principal | Installed plugin identity for a tenant | Active plugin installation in target tenant | Declared and granted capabilities plus resource permission checks | Cannot query private tables or secrets outside owned persistence/contracts. |
| Platform operator | Operational identity outside ordinary tenant roles | Not represented as tenant membership unless acting as a tenant member | Separate break-glass operational policy | No implicit business-data access; access is exceptional, time-bound, justified, and audited. |
| External member | Future customer/vendor/partner/contractor identity | Required | Restricted scoped roles | Architecture-ready but MUST remain disabled for MVP product surfaces. |

Impersonation is not an MVP capability. If introduced, it requires a new security review and an explicit permission, reason, expiry, visible indicator, original actor, effective actor, and append-only audit trail.

## 7. Role Model

### 7.1 Role Types

| Type | Ownership | Mutability | Assignment |
|---|---|---|---|
| Reserved system role | Core | Name, key, and protected permission baseline are versioned by Core | Tenant members only; assignment requires `authorization.role_assignment.manage`. |
| Plugin role template | Owning plugin | Versioned by plugin; instantiated or synchronized per tenant | Available only while the plugin is enabled. |
| Tenant custom role | Tenant | Tenant administrators may create, rename, archive, and change permissions | Cannot contain unregistered permissions or exceed the assigning administrator's grant boundary. |
| Service role | Core or owning module | Restricted to non-human principals | Cannot be assigned to human memberships. |

Roles are tenant-local except the registered definitions of reserved role templates. A role never inherits another role in v1.0. Flat permission composition avoids hidden privilege inheritance and cycle handling.

### 7.2 Baseline Human Roles

| Role key | Purpose | Default assignment scope |
|---|---|---|
| `tenant_administrator` | Tenant configuration, security administration, plugin lifecycle, and company-wide operations | Tenant |
| `management` | Cross-organizational visibility, reporting, and authorized approvals without security administration | Tenant or organization |
| `manager` | Team/department work, records, workflow, and approvals | Business unit, department, team, or location |
| `employee` | Personal work and explicitly shared operational resources | Owner, team, or department |

The role names describe baselines, not job titles. Tenants SHOULD create narrower custom roles for functions such as HR Administrator, Sales Manager, Finance Approver, CRM User, and Records Administrator.

### 7.3 Assignment Lifecycle

A role assignment has `pending`, `active`, `suspended`, `expired`, or `revoked` state. It has an effective time, optional expiry, assigning actor, reason, role, membership, tenant, scope type, and scope anchor.

Rules:

- Only `active` assignments within their effective interval contribute permissions.
- Suspending or terminating a membership immediately removes all effective assignments.
- Revoked and expired assignments remain available for audit and MUST NOT be reactivated; a new assignment is created.
- Assignment create, change, revoke, and expiry are audited.
- A principal MUST NOT grant a role/permission at a broader scope than the principal can administer.
- A principal MUST NOT assign a permission the principal cannot grant. The reserved tenant-administrator bootstrap follows a controlled tenant-provisioning path.
- Self-assignment of privileged roles is prohibited.
- Privileged assignment changes SHOULD require recent authentication; tenant-administrator assignment SHOULD support two-person approval when configured.
- Role permission changes affect active assignments immediately and invalidate authorization caches.

## 8. Permission Model

### 8.1 Naming

Permission keys use lowercase ASCII segments:

`domain.resource.action`

- `domain` identifies Core domain or plugin namespace.
- `resource` is singular snake_case when multiple words are required.
- `action` is a stable verb from the catalog.

Examples:

- `work.task.read`
- `workflow.approval.decide`
- `crm.contact.export`
- `sales.sales_order.approve`
- `hr.employee.compensation_read`

### 8.2 Action Semantics

| Action | Meaning |
|---|---|
| `create` | Create a new resource within an authorized scope. |
| `read` | Read one resource and non-sensitive fields. |
| `list` | Enumerate resources; every returned row must independently match authorization. |
| `update` | Change ordinary mutable fields. |
| `archive` | Remove from active use without destroying required history. |
| `restore` | Return an archived resource to active use. |
| `delete` | Irreversibly delete where domain and retention rules permit. |
| `manage` | Administer a bounded resource configuration; never implies every action on child resources. |
| `assign` | Change responsibility, assignee, or owner. |
| `transition` | Request an allowed workflow or lifecycle transition. |
| `approve` | Approve a business object where approval is a resource action. |
| `decide` | Submit an approval decision on an assigned Approval resource. |
| `delegate` | Delegate an assigned approval or work responsibility. |
| `publish` | Publish an immutable/versioned definition. |
| `execute` | Run an automation, integration, or controlled operation. |
| `export` | Extract data outside normal interactive reads. |
| `import` | Bulk ingest data. |
| `download` | Retrieve file binary content. |
| `upload` | Upload binary content subject to validation. |
| `install` / `enable` / `disable` / `upgrade` / `uninstall` | Explicit plugin lifecycle operations. |
| `permission_manage` | Change a role's permission set. |
| `compensation_read` / `compensation_manage` | Access specially classified HR compensation data. |

`manage` is not a wildcard. Callers MUST check the exact action permission listed by the matrix.

### 8.3 Registration and Lifecycle

- Core and each plugin own their permission namespace.
- Permissions MUST be declared with owner module, resource, action, supported scopes, sensitivity, and lifecycle status.
- Unknown, disabled, or retired permission keys fail closed.
- Permission keys are immutable after release. Semantic changes require a new permission key and migration.
- A plugin cannot activate its permissions until installation validation succeeds.
- Disabling a plugin makes its permissions ineffective but preserves role bindings for a reversible re-enable.
- Uninstall behavior MUST preserve audit references and follow plugin retention policy.
- Wildcard permissions such as `sales.*` or `*.read` MUST NOT be stored or evaluated in v1.0.
- Runtime deny grants are intentionally excluded. Exceptions are modeled as narrower roles/scopes or contextual policy; introducing explicit deny requires a new ADR because precedence becomes security-critical.

## 9. Scope Model

### 9.1 Scope Types

| Scope | Anchor | Matches |
|---|---|---|
| `tenant` | Target tenant | Every resource in the same tenant, subject to permission and contextual policy. |
| `organization` | Organization ID | Resource directly assigned to the organization or to a descendant business unit/department/team. |
| `business_unit` | Business Unit ID | Resource assigned to that unit or a descendant department/team. |
| `department` | Department ID | Resource assigned to that department or a descendant team. |
| `team` | Team ID | Resource assigned to that team. |
| `location` | Location ID | Resource explicitly classified to that location; organizational ancestry does not imply location. |
| `owner` | Contextual membership/user | Resource whose normalized owner membership equals the requesting membership, or personal work explicitly assigned to it. |
| `record` | Resource type + resource ID | Exactly one resource. Record scope does not imply access to related resources. |

### 9.2 Matching Rules

1. The resource tenant MUST equal the resolved request tenant and assignment tenant.
2. The permission MUST explicitly support the assignment's scope type.
3. The trusted resource context MUST contain the attributes required to evaluate that scope.
4. Organizational descendant matching uses the current validated hierarchy, not client-supplied paths.
5. Scope breadth does not create permission precedence. Matching assignments combine by union.
6. A location assignment matches only explicit resource location classification.
7. Owner scope is supported only for resources whose matrix row declares owner matching.
8. Record scope requires exact resource type and identifier matching in the same tenant.
9. Related-resource access is never transitive unless the matrix explicitly defines a parent-context rule.
10. If a resource has multiple qualifying teams/owners, any matching association grants access only when that association type is declared authorization-relevant by the owning module.
11. Moving a resource or member between organizational scopes takes effect transactionally and invalidates relevant authorization caches.
12. If hierarchy, ownership, plugin state, or assignment context is missing or stale beyond the accepted consistency boundary, authorization fails closed.
13. Create checks evaluate the proposed parent/organizational context before persistence; subsequent read/update checks evaluate persisted context.

### 9.3 Supported Scope Sets

The matrices use these abbreviations:

- `T`: tenant
- `O`: organization
- `BU`: business unit
- `D`: department
- `TM`: team
- `L`: location
- `OWN`: owner
- `R`: record
- `—`: not scope-applicable or platform-controlled

## 10. Evaluation Contract

Authorization evaluation MUST follow this order:

```text
authorize(principal, tenant, permission, resource_context, request_context):
  require authenticated(principal)
  require active_session_or_credential(principal)
  require known_active_permission(permission)

  if resource_context is tenant_owned:
    require resource_context.tenant_id == tenant.id
    require active_membership(principal, tenant)

  if principal is plugin:
    require active_plugin_installation(principal.plugin, tenant)
    require declared_and_granted_capability(principal.plugin, permission)

  assignments = active_role_assignments(principal.membership, tenant)
  candidates = assignments granting exact(permission)
  require any candidate whose scope matches trusted(resource_context)

  require resource_state_allows(permission.action)
  require field_classification_allows(permission, requested_fields)
  require workflow_and_separation_of_duties_allow(request_context)
  require step_up_authentication_if_required(permission, request_context)

  allow and emit required audit/security telemetry
```

Existence-hiding endpoints SHOULD return a not-found-equivalent response when revealing resource existence would leak sensitive information. Internally, denial reason codes remain distinguishable for audit and diagnostics.

Authorization caches MAY contain registered permissions, role-permission mappings, and stable hierarchy closures. They MUST be tenant-keyed, bounded by short expiry, invalidated on relevant changes, and never turn an indeterminate result into allow.

## 11. Baseline Role Legend

Matrices use:

- `TA`: Tenant Administrator
- `MG`: Management
- `MR`: Manager
- `EE`: Employee
- `C`: Tenant custom role expected for domain-sensitive access
- `S`: Service/plugin principal with explicit capability

A listed role is a recommended baseline eligible to receive the permission; it is not an automatic grant. Actual access always requires an active assignment and matching scope.

## 12. Core Resource Permission Matrix

`Audit` values:

- `A`: append-only audit event required for successful action and material denial where specified
- `S`: security event required
- `O`: ordinary activity/observability; audit when material
- `—`: no dedicated audit beyond standard request telemetry

### 12.1 Identity and Organization

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit | Invariants |
|---|---|---|---|---|---|---|---|
| `identity.user` | `read`, `update` | TA, EE(self) | T, OWN, R | Tenant access is mediated through membership; self-update is limited to approved profile fields. | Credential and MFA secrets excluded; email/security changes require recent auth. | A/S | INV-002, INV-003, INV-005 |
| `identity.session` | `list`, `revoke` | TA, EE(self) | T, OWN, R | EE sees/revokes own sessions; TA may revoke tenant-member sessions, not inspect secrets. | Session token never returned; revoke is immediate. | S | INV-002, INV-003, INV-005 |
| `identity.credential` | `manage` | EE(self), controlled identity service | OWN | No tenant role permits credential material reads. | Recent auth and secure recovery policy required. | S | INV-003, INV-005 |
| `identity.mfa_factor` | `manage` | EE(self), controlled identity admin flow | OWN, R | Admin reset is separate from reading/enrolling a factor. | Recent auth; recovery/reset is high-risk. | S | INV-003, INV-005 |
| `organization.tenant` | `read`, `update` | TA, MG(read) | T | One current tenant only. | Billing/security configuration fields require narrower permissions. | A | INV-001–003 |
| `organization.organization` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR(read) | T, O, R | Hierarchy and tenant are immutable through ordinary update paths. | Structural changes require cycle and impact validation. | A | INV-001–003 |
| `organization.business_unit` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR(read) | T, O, BU, R | Must belong to one organization in the same tenant. | Moving scope requires impact preview. | A | INV-001–003 |
| `organization.department` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR(read) | T, O, BU, D, R | Must belong to valid same-tenant hierarchy. | Moving scope requires impact preview. | A | INV-001–003 |
| `organization.team` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR | T, O, BU, D, TM, R | May have members and authorization-relevant work links. | Membership impact must be evaluated. | A | INV-001–003 |
| `organization.position` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR(read) | T, O, BU, D, R | Position grants no security role implicitly. | Prevent role/position conflation. | A | INV-002, INV-003 |
| `organization.location` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR(read) | T, O, L, R | Location hierarchy is independent from organization scope. | Moving resources requires reauthorization. | A | INV-001–003 |
| `organization.cost_center` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, C | T, O, BU, D, R | Financial classification only; does not grant data access. | Custom role recommended. | A | INV-001–003 |
| `organization.membership` | `invite`, `read`, `list`, `update`, `suspend`, `terminate` | TA, MR(read within scope), C | T, O, BU, D, TM, R | Membership is unique per user and tenant; manager reads only managed scope. | Invite, suspend, terminate require exact actions and recent auth for admins. | A/S | INV-001–003 |
| `organization.team_membership` | `create`, `read`, `list`, `update`, `archive` | TA, MR | T, D, TM, R | Both membership and team must be same tenant. | Changes can alter effective resource access. | A/S | INV-001–003 |

The canonical permission keys are formed directly from each resource and action, for example `organization.department.update` and `identity.session.revoke`. `organization.membership.invite`, `suspend`, and `terminate` are explicit registered actions.

### 12.2 Authorization and Configuration

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit | Invariants |
|---|---|---|---|---|---|---|---|
| `authorization.role` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, C(read) | T, R | Reserved roles cannot be renamed/archived by tenants. | Grant-boundary check; role changes invalidate caches. | A/S | INV-002–004 |
| `authorization.role_permission` | `read`, `permission_manage` | TA | T, R | Exact permissions only; no wildcards or unknown keys. | Actor cannot grant beyond own grant boundary. | A/S | INV-002–004 |
| `authorization.role_assignment` | `create`, `read`, `list`, `update`, `revoke` | TA, delegated C | T, O, BU, D, TM, L, R | Assignment has exactly one scope; self-privilege escalation prohibited. | Recent auth; optional two-person control for privileged roles. | A/S | INV-002–004 |
| `authorization.permission` | `read`, `list` | TA, C | T | Registry is Core/plugin owned and tenant-readable for role design. | Tenant actors cannot create or mutate permission definitions. | O | INV-003, INV-004, INV-026 |
| `configuration.tenant_setting` | `read`, `update` | TA, C | T, R | Secret values use a dedicated secret reference, not plaintext settings. | Sensitive settings masked; security settings need recent auth. | A/S | INV-005, INV-031 |
| `configuration.feature` | `read`, `manage` | TA | T, R | Only supported tenant-configurable features. | Cannot disable security invariants. | A | INV-003, INV-032 |
| `integration.connection` | `create`, `read`, `list`, `update`, `archive`, `execute` | TA, C, S | T, O, R | Secret material is write-only and stored outside ordinary configuration. | SSRF/egress policy, credential isolation, recent auth. | A/S | INV-002, INV-005, INV-031 |
| `integration.webhook` | `create`, `read`, `list`, `update`, `archive`, `execute` | TA, C, S | T, O, R | Delivery endpoint and subscribed events are tenant-bound. | URL validation, signing secret masking, SSRF control. | A/S | INV-002, INV-005, INV-018 |

### 12.3 Core Data and Search

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit | Invariants |
|---|---|---|---|---|---|---|---|
| `data.object_type` | `create`, `read`, `list`, `update`, `archive`, `restore`, `publish` | TA, C | T, O, R | Core/plugin-owned types have protected definitions; custom types are tenant-owned. | Published schema changes require compatibility validation. | A | INV-009–012 |
| `data.field_definition` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, C | T, O, R | Access is bounded by parent object type. | Classification/visibility changes trigger security review. | A/S | INV-007, INV-010 |
| `data.record` | `create`, `read`, `list`, `update`, `archive`, `restore`, `delete`, `export`, `import` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Object type declares authorization-relevant org, team, location, and owner fields. | Field-level classification filters reads/writes; delete only when retention permits. | A for mutation/export/import | INV-001–003, INV-009–012 |
| `data.relationship` | `create`, `read`, `list`, `archive`, `restore` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | Actor must read both endpoints and mutate the owning endpoint; endpoints must share tenant. | Relationship type controls legal source/target types. | A | INV-001–003, INV-012 |
| `search.query` | `execute` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN | Search is a query capability, never an access bypass; each result is filtered by its resource permission. | Queries and snippets must not reveal unauthorized fields/existence. | O/S for anomalous use | INV-002, INV-007 |
| `search.saved_query` | `create`, `read`, `list`, `update`, `archive` | TA, MG, MR, EE, C | T, O, D, TM, OWN, R | Personal by default; sharing requires target-scope access. | Stored filters cannot embed secrets. | O | INV-002, INV-007 |

Field-level rules refine resource authorization and cannot grant access independently. An object/field definition MUST declare field classification such as `public_internal`, `confidential`, `restricted`, or a domain-specific permission key. Unauthorized fields are omitted or rejected consistently; write attempts to unauthorized fields fail rather than being silently ignored.

### 12.4 Work, Workflow, and Approval

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit | Invariants |
|---|---|---|---|---|---|---|---|
| `work.task` | `create`, `read`, `list`, `update`, `assign`, `transition`, `archive`, `restore` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | OWN matches assignee or explicitly declared owner, not reporter by default. | State transition and assignment checks are separate. | A for assign/transition | INV-001–003, INV-021, INV-022 |
| `work.project` | `create`, `read`, `list`, `update`, `assign`, `transition`, `archive`, `restore`, `export` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Membership in a project does not imply unrelated resource access. | Export is explicit. | A | INV-001–003, INV-021 |
| `work.milestone` | `create`, `read`, `list`, `update`, `transition`, `archive` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | Parent project context may narrow but never broaden access. | Transition follows project/workflow rules. | A | INV-014, INV-015, INV-022 |
| `work.request` | `create`, `read`, `list`, `update`, `assign`, `transition`, `archive`, `restore` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Requester has OWN read where declared; fulfillment ownership is explicit. | Sensitive request types may require custom permission/field rules. | A | INV-001–003, INV-021, INV-022 |
| `work.assignment` | `create`, `read`, `list`, `update`, `revoke` | TA, MR, C, S; EE self-read | T, O, BU, D, TM, OWN, R | Actor needs assign permission on parent resource; assignee must be eligible in scope. | Assignment cannot be used to escape parent access. | A | INV-002, INV-021 |
| `workflow.definition` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, C | T, O, R | Owning module/resource type is immutable after use. | Draft definitions only are mutable. | A | INV-012, INV-013 |
| `workflow.version` | `create`, `read`, `list`, `publish` | TA, C | T, O, R | Published versions are immutable; one selected active version per binding. | Publish validates permissions, states, conditions, and actions. | A/S | INV-013, INV-014 |
| `workflow.instance` | `read`, `list`, `transition` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Access follows bound resource plus explicit workflow permission. | Expected current state/version required. | A | INV-014, INV-015 |
| `workflow.approval` | `create`, `read`, `list`, `decide`, `delegate`, `cancel` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | OWN means currently assigned approver; creation/cancel requires parent workflow authority. | Decision actor must be authorized at decision time; no self-approval when SoD rule applies. | A/S | INV-015, INV-023 |
| `workflow.delegation` | `create`, `read`, `list`, `revoke` | TA, MG, MR, EE(self), C | T, O, BU, D, TM, OWN, R | Delegator can delegate only eligible authority and bounded dates/scope. | Cannot delegate prohibited decisions or broaden scope. | A/S | INV-003, INV-023 |

Possessing `workflow.instance.transition` does not guarantee a transition. The transition definition may additionally require an exact domain permission such as `sales.sales_order.approve`, validation, current state, approval completion, and separation-of-duties checks.

### 12.5 Content and Communication

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit | Invariants |
|---|---|---|---|---|---|---|---|
| `content.document` | `create`, `read`, `list`, `update`, `archive`, `restore`, `export` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Parent folder/resource policy may narrow access; no implicit access from URL possession. | Classification and export checks apply. | A | INV-001–003, INV-028 |
| `content.document_version` | `create`, `read`, `list`, `download` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Read requires parent document access; published versions are immutable. | Binary access uses authorized short-lived delivery. | A for download when sensitive | INV-028–030 |
| `content.file` | `upload`, `read`, `download`, `archive`, `delete` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | File must be attached to or owned by an authorized resource; unattached uploads are quarantined. | Validation, malware status, random object key, retention. | A/S | INV-028–030 |
| `content.folder` | `create`, `read`, `list`, `update`, `archive`, `restore` | TA, MG, MR, EE, C | T, O, BU, D, TM, L, OWN, R | Folder inheritance may narrow child visibility; it never grants broader child access than resource policy. | Moving folders requires descendant reauthorization. | A | INV-002, INV-028 |
| `content.attachment` | `create`, `read`, `list`, `archive` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | Actor must read file and mutate target resource; same tenant required. | Attachment does not copy source access to target viewers automatically. | A | INV-001–003, INV-028 |
| `communication.comment` | `create`, `read`, `list`, `update`, `archive` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | Requires read access to parent; update/archive own comment unless moderation permission is granted. | Parent sensitivity applies; edited content history retained where required. | O/A for moderation | INV-002, INV-006 |
| `communication.mention` | `create`, `read` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | Mentioned principal must be same tenant and eligible to discover parent context. | Mention never grants parent access. | O | INV-002, INV-007 |
| `communication.notification` | `read`, `list`, `update` | TA, MG, MR, EE, C | OWN, R | Strictly recipient-owned; admins do not read notification content by role alone. | Payload contains no secrets and no unauthorized snapshot. | O | INV-002, INV-007 |
| `communication.subscription` | `create`, `read`, `list`, `update`, `archive` | TA, MG, MR, EE, C | OWN, R | Principal can subscribe only to resources/events it can access. | Delivery rechecks authorization where content can change sensitivity. | O | INV-002, INV-007 |

### 12.6 Automation, Events, Audit, and Plugin Lifecycle

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit | Invariants |
|---|---|---|---|---|---|---|---|
| `automation.rule` | `create`, `read`, `list`, `update`, `archive`, `restore`, `enable`, `disable` | TA, C | T, O, BU, D, TM, R | Rule execution authority is bounded to declared scopes and service identity. | Creator cannot encode actions beyond grant boundary. | A/S | INV-003, INV-008, INV-019 |
| `automation.execution` | `read`, `list`, `execute`, `cancel`, `retry` | TA, C, S | T, O, BU, D, TM, R | Manual execution/retry uses exact rule and target context. | Idempotency required; payload secrets masked. | A | INV-008, INV-019, INV-020 |
| `event.domain_event` | `read`, `list` | TA, C, S | T, O, R | Human access is operationally restricted; consumers receive declared event types only. | Payload field classification and tenant filtering mandatory. | O/S | INV-016–020 |
| `event.outbox_message` | `read`, `list`, `retry` | TA, C, S | T, R | Operational resource; ordinary members have no access. | Retry is idempotency-aware; payload immutable. | A/S | INV-016–020 |
| `audit.activity` | `read`, `list` | TA, MG, MR, EE, C | T, O, BU, D, TM, OWN, R | Activity visibility follows subject-resource access. | Sensitive details redacted. | — | INV-002, INV-007 |
| `audit.audit_event` | `read`, `list`, `export` | TA, designated Auditor C | T, O, R | No update/archive/delete permission exists. Subject-resource sensitivity may require redaction. | Export requires recent auth and reason. | S/A | INV-005, INV-006 |
| `audit.security_event` | `read`, `list`, `export` | TA, designated Security Auditor C | T, R | No ordinary manager/employee access; platform security events may be outside tenant surface. | Highly restricted; export monitored. | S/A | INV-005, INV-006 |
| `plugin.catalog_entry` | `read`, `list` | TA, MG, C | T | Availability does not imply installation permission. | Manifest/signature status visible. | O | INV-024–026 |
| `plugin.installation` | `read`, `list`, `install`, `enable`, `disable`, `upgrade`, `uninstall` | TA | T, R | One installation lifecycle per plugin and tenant. | Validate provenance, dependencies, migrations, requested capabilities, and rollback path. | A/S | INV-024–027 |
| `plugin.capability_grant` | `read`, `list`, `manage` | TA | T, R | Grants cannot exceed plugin manifest declarations or Core policy. | Capability expansion requires explicit review; no secret access. | A/S | INV-004, INV-005, INV-026 |
| `plugin.configuration` | `read`, `update` | TA, plugin-specific C | T, R | Plugin can access only its own validated configuration contract. | Secrets are references/write-only values. | A/S | INV-004, INV-005, INV-031 |

Audit, security-event, and published domain-event stores expose no ordinary mutation or deletion permissions. Retention enforcement, legal deletion, or cryptographic erasure is a separately controlled platform operation outside tenant RBAC and must preserve required accountability metadata.

## 13. CRM Plugin Permission Matrix

CRM owns business meaning for leads, contacts, accounts, opportunities, pipelines, and CRM activities. Core may store generic records, work, comments, documents, workflow, and audit, but CRM application services own CRM invariants.

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit |
|---|---|---|---|---|---|---|
| `crm.account` | `create`, `read`, `list`, `update`, `assign`, `archive`, `restore`, `export`, `import` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Owner/team/territory classification explicitly mapped by CRM. | Export/import explicit; protected fields may require custom permissions. | A |
| `crm.contact` | `create`, `read`, `list`, `update`, `assign`, `archive`, `restore`, `export`, `import` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Contact access may derive from explicit account association only when configured; never implicitly across tenant. | Personal data classification, export reason, retention. | A/S |
| `crm.lead` | `create`, `read`, `list`, `update`, `assign`, `transition`, `archive`, `restore`, `export`, `import`, `convert` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | OWN matches assigned owner; conversion creates authorized target resources atomically. | Conversion requires create permissions for all targets. | A |
| `crm.opportunity` | `create`, `read`, `list`, `update`, `assign`, `transition`, `archive`, `restore`, `export` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Pipeline/stage transition and owner checks apply. | Value/forecast fields may be confidential. | A |
| `crm.pipeline` | `create`, `read`, `list`, `update`, `archive`, `restore`, `publish` | TA, MG, C | T, O, BU, D, TM, R | Published pipeline versions constrain opportunity transitions. | Structural changes require compatibility checks. | A |
| `crm.activity` | `create`, `read`, `list`, `update`, `archive` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, OWN, R | Parent CRM resource read required; owner scope may match assigned participant. | Communication content follows parent sensitivity. | O/A |

Additional exact actions produce permission keys such as `crm.lead.convert`. CRM service principals require both capability grants and these permissions.

## 14. Sales Plugin Permission Matrix

Sales owns quotations, sales orders, line items, pricing rules, and sales-specific lifecycle rules. Cross-module fulfillment and finance reactions occur through contracts/events, not table access.

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit |
|---|---|---|---|---|---|---|
| `sales.quotation` | `create`, `read`, `list`, `update`, `assign`, `transition`, `approve`, `archive`, `restore`, `export` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Account/opportunity association does not replace explicit quotation access. | Approval may require value threshold and SoD. | A/S |
| `sales.quotation_line` | `create`, `read`, `list`, `update`, `archive` | TA, MG, MR, EE, C, S | Inherit parent quotation scope | Actor must have corresponding parent action; line access cannot exceed parent. | Pricing/discount controls apply. | A with parent |
| `sales.sales_order` | `create`, `read`, `list`, `update`, `assign`, `transition`, `approve`, `cancel`, `archive`, `export` | TA, MG, MR, EE, C, S | T, O, BU, D, TM, L, OWN, R | Approval/cancel are state-specific; originating quotation does not confer access automatically. | Threshold approval, SoD, immutable accepted commercial terms. | A/S |
| `sales.sales_order_line` | `create`, `read`, `list`, `update`, `archive` | TA, MG, MR, EE, C, S | Inherit parent sales order scope | Parent authorization and state control every action. | Restricted after order approval. | A with parent |
| `sales.pricing_rule` | `create`, `read`, `list`, `update`, `archive`, `restore`, `publish` | TA, MG, C | T, O, BU, D, TM, R | Only published applicable versions affect pricing. | High-impact; publish requires recent auth and review where configured. | A/S |
| `sales.discount` | `apply`, `approve` | TA, MG, MR, C | T, O, BU, D, TM, R | Apply and approve are separate; thresholds can require a different approver. | Requester MUST NOT approve own discount when SoD applies. | A/S |

Line-resource scopes are derived only from the authorized parent inside the same transaction. They do not receive independent owner grants. Exact permissions include `sales.discount.apply`, `sales.discount.approve`, and `sales.sales_order.cancel`.

## 15. HR Plugin Permission Matrix

HR owns employee business profiles, employment, compensation, leave, recruitment, onboarding, and HR-specific workflows. User identity remains Core-owned.

| Resource | Actions | Eligible roles | Scopes | Ownership/context rule | Sensitive controls | Audit |
|---|---|---|---|---|---|---|
| `hr.employee` | `create`, `read`, `list`, `update`, `archive`, `restore`, `export` | TA, MG(limited), MR(limited), EE(self-limited), HR C, S | T, O, BU, D, TM, L, OWN, R | OWN maps employee profile to current membership; manager access excludes restricted fields. | Field-level classification mandatory; export highly restricted. | A/S |
| `hr.employment` | `create`, `read`, `list`, `update`, `transition`, `archive` | TA, HR C, MG(limited), MR(limited), EE(self-read) | T, O, BU, D, L, OWN, R | Effective-dated employment; organizational scope evaluated at relevant date. | Contract and termination data restricted. | A/S |
| `hr.compensation` | `create`, `compensation_read`, `compensation_manage`, `archive` | HR C, narrowly delegated MG | T, O, BU, D, OWN, R | No baseline TA/MR/EE access; self-service compensation requires explicit custom grant and product policy. | Restricted data, recent auth, export excluded by default, all reads auditable. | A/S |
| `hr.leave_request` | `create`, `read`, `list`, `update`, `transition`, `approve`, `cancel` | TA, HR C, MR, EE(self), S | T, O, BU, D, TM, OWN, R | Employee owns request; approver authority follows hierarchy/workflow and SoD. | Sensitive leave categories are redacted from ordinary managers where required. | A/S |
| `hr.requisition` | `create`, `read`, `list`, `update`, `transition`, `approve`, `archive` | TA, HR C, MG, MR | T, O, BU, D, TM, L, OWN, R | Hiring manager/HR ownership explicit. | Headcount/budget approval rules may apply. | A |
| `hr.candidate` | `create`, `read`, `list`, `update`, `transition`, `archive`, `delete`, `export`, `import` | HR C, designated interviewer C, S | T, O, BU, D, TM, OWN, R | Interviewers receive record-scoped or requisition-scoped limited access. | Personal data, consent, retention and deletion policy mandatory. | A/S |
| `hr.application` | `create`, `read`, `list`, `update`, `transition`, `archive` | HR C, designated interviewer C, S | T, O, BU, D, TM, OWN, R | Access follows requisition plus explicit candidate/application assignment. | Notes/ratings visibility separated by field policy. | A/S |
| `hr.onboarding` | `create`, `read`, `list`, `update`, `assign`, `transition`, `archive` | TA, HR C, MR, EE(self-limited), S | T, O, BU, D, TM, L, OWN, R | New employee sees only assigned/self-visible steps; HR owns process. | Documents and tasks retain their own authorization. | A |

Tenant Administrator is not a universal HR-data reader. Administrative control and business-data access remain separable. Emergency access to restricted HR data requires a dedicated custom role or future break-glass control, reason, expiry, and audit.

## 16. Special Authorization Policies

### 16.1 Role and Permission Administration

- Role administration and permission-set administration are separate permissions.
- The system computes the administrator's grant boundary from exact permissions and maximum scopes the actor is authorized to administer.
- Editing a role MUST reject any permission/scope combination outside that boundary.
- An actor cannot approve or complete the actor's own privileged-role assignment when two-person control is enabled.
- The last active tenant administrator cannot be removed without a validated replacement or controlled tenant closure process.

### 16.2 Approvals and Delegation

- Approval assignment identifies eligible actors, but `workflow.approval.decide` and all domain permissions remain required.
- A decision captures original assignee, effective decision actor, delegation chain, permission snapshot reference, result, reason, and timestamp.
- Delegation cannot broaden permission, scope, amount threshold, resource type, or effective period.
- Separation-of-duties policies may prohibit requester, creator, owner, or prior actor from deciding.
- Reassignment, delegation, expiry, escalation, decision, cancellation, and override are audited.

### 16.3 Workflow Transitions

A transition is allowed only when all pass:

1. resource read access;
2. `workflow.instance.transition` at matching scope;
3. transition-specific domain permission when declared;
4. expected current state/version;
5. transition conditions and validation;
6. required approvals;
7. separation-of-duties and step-up rules.

### 16.4 Search, List, Reports, and Dashboards

- `list` and `search.query.execute` authorize query execution, not unrestricted rows.
- Query predicates MUST include tenant and effective resource scope before pagination and aggregation.
- Counts, facets, suggestions, snippets, exports, and dashboards MUST NOT reveal unauthorized resources or fields.
- Derived read models carry enough authorization attributes to enforce current access, or they join back to an authoritative access projection.
- Permission/scope changes invalidate or safely age out derived authorization projections.

### 16.5 Bulk Operations, Import, and Export

- Bulk operations evaluate each target resource; one broad request does not bypass row checks.
- Atomic bulk operations fail completely on any denial. Best-effort operations return per-item outcomes without leaking unauthorized existence.
- Import requires `import` plus `create`/`update` as applicable and validates proposed scope/ownership.
- Export requires explicit `export`, applies field redaction, records filters and reason where sensitive, and produces a protected expiring artifact.
- Exported files retain classification and download authorization.

### 16.6 Background Jobs and Automation

- Jobs carry tenant ID, initiating actor where applicable, service principal, correlation/causation IDs, requested capability, and trusted target references.
- Delayed execution re-evaluates current permissions for actor-authority operations. System-owned maintenance uses a narrowly defined service capability instead.
- Missing tenant context, revoked capability, disabled plugin, or indeterminate resource context fails the job closed and retains observable failure state.
- Retries are idempotent and never silently switch to elevated service authority.

### 16.7 Plugin and Cross-Module Access

Cross-module access requires all of:

1. active and valid plugin installation;
2. manifest-declared capability;
3. tenant-approved capability grant when required;
4. exact resource permission for the plugin/service principal;
5. matching tenant and scope;
6. call through the owning module's public application contract.

A plugin MUST NOT read or mutate another module's private tables. Event subscriptions are limited to declared event types and authorized payload fields. Receiving an event does not create authority to fetch the referenced resource later.

### 16.8 Files and Attachments

- Upload permission creates quarantined metadata only until configured validation completes.
- Download requires current authorization against file classification and attached/owning resource.
- Pre-signed URLs are short-lived, tenant-bound through server authorization, non-enumerable, and never treated as durable grants.
- Attachment creation requires source file access and target mutation rights; attachment visibility does not weaken either policy.

### 16.9 Audit and Security Events

- No tenant permission exists to update, archive, restore, or delete audit/security events.
- Sensitive values are redacted before persistence.
- Successful privileged actions are audited; significant denials, privilege escalation attempts, cross-tenant attempts, and break-glass operations create security events.
- Audit reads and exports are themselves audited.

## 17. Decision Examples

### 17.1 Allowed: Department Manager Updates Team Task

A manager has `work.task.update` through a role assignment scoped to Department Sales. The task belongs to a descendant Sales team in the same tenant. The task state permits updates. Access is allowed.

### 17.2 Denied: Cross-Tenant Record ID

An employee has `data.record.read` at tenant scope in Tenant A but requests a record whose trusted context belongs to Tenant B. Evaluation stops at tenant comparison. Access is denied regardless of matching role or record identifier.

### 17.3 Allowed: Employee Reads Own Leave Request

An employee has `hr.leave_request.read` at owner scope. The request's employee profile resolves to the same active membership. Access is allowed, subject to field policy.

### 17.4 Denied: Manager Reads Compensation

A department manager has broad `hr.employee.read` but lacks `hr.compensation.compensation_read`. Compensation is a separate restricted resource. Access is denied.

### 17.5 Denied: Self-Approval

A sales manager has `sales.discount.approve` and a matching department scope, but the manager requested the discount and the active policy requires separation of duties. Contextual policy denies the decision.

### 17.6 Denied: Plugin with Permission but No Capability

A CRM plugin service role contains `work.task.create`, but its active manifest/grant has no task-creation capability. Plugin capability evaluation fails before resource scope evaluation.

### 17.7 Denied: Background Job with Revoked Authority

A delayed export job carries the initiating actor and tenant context. Before execution, the actor's export role is revoked. Re-evaluation denies execution; the job records a non-retryable authorization failure.

### 17.8 Allowed: Record-Scoped Interviewer

An interviewer receives a custom role containing `hr.candidate.read` and `hr.application.read`, assigned to the exact application record. Only that record and approved fields are visible; related employee, compensation, and other candidate records remain inaccessible.

## 18. Audit Requirements

Every authorization audit record for a material action SHOULD include:

- tenant ID;
- actor principal and effective membership;
- original actor and service/plugin principal where applicable;
- action and exact permission key;
- resource type and stable resource ID;
- matched role assignment and scope type/anchor;
- result and reason code;
- request, correlation, and causation IDs;
- source context and authentication assurance where safe;
- before/after values where material and safe;
- timestamp and policy/permission catalog version.

Ordinary high-volume successful reads are not universally audited. Reads of restricted HR data, credentials/security administration surfaces, audit/security events, secrets metadata, and sensitive exports MUST be audited. Denials are sampled for ordinary noise but MUST be retained for cross-tenant attempts, privileged actions, repeated enumeration, or policy-defined security signals.

## 19. Security Verification Requirements

### 19.1 Unit and Policy Tests

- Exact permission match; unknown, retired, and wildcard keys deny.
- Inactive membership, role, assignment, plugin, session, or service credential denies.
- Assignment effective/expiry boundaries are deterministic.
- Every scope type has positive, negative, sibling, ancestor, descendant, and moved-resource tests.
- Multiple assignments union correctly without widening tenant or unsupported scopes.
- Owner and record scopes do not grant related-resource access.
- Role grant-boundary and self-escalation checks deny correctly.
- Field classification cannot broaden resource access.
- Workflow state, approval, delegation, threshold, and SoD policies compose with RBAC.

### 19.2 Integration Tests

- Every tenant-owned endpoint rejects cross-tenant identifiers, associations, list filters, bulk targets, exports, search, and attachment paths.
- List counts, facets, pagination, reports, dashboards, and search snippets contain only authorized data.
- Background jobs preserve tenant and actor context and fail closed after revocation.
- Plugin calls require active installation, capability, exact permission, scope, and public contract.
- Cache invalidation follows role, assignment, hierarchy, ownership, membership, and plugin-state changes.
- Audit records are append-only and capture required privileged actions and denials.
- File upload/download, pre-signed delivery, attachment, and malware/quarantine states enforce current policy.

### 19.3 Matrix Completeness Tests

- Every API/application action maps to exactly one primary permission and optional declared contextual policies.
- Every registered permission maps to an owning module and at least one tested use case or is explicitly reserved.
- Every resource identifies supported scopes and trusted authorization attributes.
- No plugin permission uses the Core namespace.
- No role contains wildcard or unknown permissions.
- No restricted field is returned under ordinary `read` without its additional permission.

## 20. Traceability

| Requirement | Specification control |
|---|---|
| ADR-003 shared tenant boundary | Tenant-first evaluation, same-tenant scope rules, negative cross-tenant tests. |
| ADR-004 User separated from Employee | `identity.user` and `hr.employee` are separate resources and permissions. |
| ADR-005 RBAC + resource permissions + scopes | Roles, exact permission catalog, scoped assignments, trusted resource context. |
| ADR-006 Core/plugin boundary | Separate namespaces and ownership; public contracts for cross-module access. |
| ADR-007 explicit plugin capabilities | Plugin principal requires installation, capability grant, permission, and scope. |
| ADR-009 workflow checks and approvals | Layered transition, approval, delegation, and SoD policies. |
| ADR-010 tenant-scoped events | Event consumers are capability-limited; event payloads and fetches remain authorized. |
| ADR-011 append-only audit | No mutation permissions; privileged access and changes audited. |
| INV-001–004 | Tenant-first fail-closed server authorization and plugin restrictions. |
| INV-005–008 | Secret redaction, immutable audit, authorized search, contextual background jobs. |
| INV-009–015 | Stable resource identity, definition validation, module boundaries, immutable workflows, authorized transitions. |
| INV-016–020 | Tenant-aware event permissions, versioned payloads, idempotent execution/outbox operations. |
| INV-021–023 | Explicit work ownership, transition control, attributable authorized approvals. |
| INV-024–027 | Declared plugin dependencies, migrations, permissions, and audited lifecycle. |
| INV-028–030 | Authorized file access, quarantine/validation, non-predictable object delivery. |
| MVP permission acceptance | Negative authorization and tenant-isolation test requirements are mandatory. |

## 21. Explicit v1.0 Decisions

1. Authorization uses explicit allow and default deny; runtime deny grants are excluded.
2. Wildcard permissions are excluded from storage and evaluation.
3. Roles are flat permission sets with no role inheritance.
4. Each role assignment binds one role to one membership at exactly one scope.
5. Matching assignments combine by union but cannot cross tenants.
6. Organizational scope supports descendant matching; location is independent; owner and record scopes are exact.
7. Tenant Administrator is powerful but not a bypass for restricted HR data, credentials, audit immutability, plugin boundaries, or domain invariants.
8. Field classification can narrow authorized reads/writes but cannot grant resource access.
9. Plugin principals require both capability authorization and ordinary resource authorization.
10. Background operations preserve tenant context and re-evaluate actor authority where the operation represents that actor.
11. Exports, imports, bulk operations, approval decisions, and sensitive HR reads use explicit permissions.
12. The External Member principal remains architecture-ready but disabled for MVP surfaces.

## 22. Deferred Decisions

The following are intentionally deferred to later engineering packages or require a new ADR if introduced:

- platform-operator break-glass implementation and support-access workflow;
- explicit deny grants and their precedence;
- role inheritance;
- public external-member/customer/vendor portal roles;
- attribute-based policy language beyond declared contextual policies;
- cross-tenant collaboration or shared records;
- legal retention/erasure policy by jurisdiction;
- database Row-Level Security implementation details;
- dedicated policy engine adoption;
- field-encryption and key-management architecture.

These deferrals do not block Gate 1 approval because v1.0 fails closed and defines implementation boundaries for the MVP.

## 23. Approval Gate

| Gate | Artifact | Status | Approval date | Approver | Notes |
|---|---|---|---|---|---|
| Gate 1 | RBAC + Resource Permission Matrix v1.0 | Approved | 2026-08-29 | Product owner | Approved in conversation with `APPROVED - RBAC v1.0`. |
| Gate 2 | Complete ERD v1.0 | Approved | 2026-08-29 | Product owner | Approved in conversation with `APPROVED — Complete ERD v1.0`. |
| Gate 3 | PostgreSQL Schema v1.0 | In progress | — | — | Authorized by approved Gate 2. |

Approval phrase: `APPROVED — RBAC v1.0`
