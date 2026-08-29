# CompanyOS Complete ERD v1.0

Status: Approved — Gate 2
Date: 2026-08-29
Approved: 2026-08-29
Scope: Core Platform, CRM, Sales, HR
Normative authorization input: `04-engineering/security/rbac-resource-permission-matrix-v1.0.md` (Approved Gate 1)

## 1. Purpose

This document defines the complete logical data model for CompanyOS v1.0. It covers canonical transactional entities, required association/version/history entities, derived operational projections, tenant boundaries, module ownership, lifecycle, retention, and cross-domain references.

This ERD is normative for the PostgreSQL Schema. Physical PostgreSQL types, indexes, RLS, triggers, database roles, and migration implementation are deferred to Gate 3.

## 2. Normative References

- CompanyOS Product Blueprint v1.0
- CompanyOS Architecture Overview v1.0
- ADR-003 through ADR-013
- CompanyOS Core Domain Model v1.0
- CompanyOS Domain Boundaries v1.0
- CompanyOS Domain Invariants v1.0
- CompanyOS RBAC and Resource Permission Matrix v1.0 (Approved Gate 1)

## 3. Global Modeling Conventions

### 3.1 Identifiers and Tenancy

- Every durable entity has an opaque, globally unique `id`.
- Every tenant-owned entity carries `tenant_id`, either directly or through an immutable identifying parent where explicitly marked `parent-bound`.
- Tenant-owned foreign keys MUST enforce same-tenant references in the physical schema.
- Human-readable keys such as permission key, plugin key, object type key, document number, and employee number are alternate keys, not primary keys.
- Cross-tenant foreign keys and cross-tenant business relationships are prohibited.
- Platform catalog entities are marked `platform`; all others are `tenant` or `parent-bound`.

### 3.2 Common Columns

Unless explicitly excluded, mutable canonical entities contain:

- `id`
- `tenant_id` when tenant-owned
- `created_at`, `created_by`
- `updated_at`, `updated_by`
- `version` for optimistic concurrency

Lifecycle entities additionally contain `status` and relevant effective timestamps. Archivable entities contain `archived_at` and `archived_by`; they are not physically deleted during ordinary operations.

Immutable event, audit, published-version, decision, revision, and history entities contain creation/occurrence metadata but no ordinary update metadata.

### 3.3 Resource Registry

`core.resource` is the integrity anchor for generic CompanyOS resources. Each securable canonical aggregate root has exactly one resource row with:

- tenant;
- registered resource type;
- owning module;
- concrete aggregate ID;
- authorization projection attributes;
- lifecycle state and version.

Generic features such as comments, attachments, activities, workflow bindings, audit subjects, subscriptions, and cross-module relationships reference `resource_id`, not an unconstrained `(resource_type, resource_id)` pair.

Rules:

1. `core.resource_type` defines one owner module, permission namespace, and projection contract.
2. `(tenant_id, resource_type_id, concrete_id)` is unique.
3. The owning module MUST create/update the resource row transactionally with the concrete aggregate.
4. A resource row cannot exist without a registered type and owning aggregate; physical enforcement uses module-specific registration constraints/triggers defined at Gate 3.
5. Authorization projection attributes accelerate filtering but are derived from canonical ownership/scope data and MUST be transactionally synchronized or fail closed.
6. Generic references do not authorize access; approved RBAC evaluation still applies.

### 3.4 Canonical, Immutable, and Derived Data

| Class | Meaning | Mutation rule |
|---|---|---|
| `C` Canonical | System-of-record business/configuration data | Controlled create/update/archive through owner module. |
| `I` Immutable | Published version, event, audit, decision, revision, or history fact | Insert-only after creation/publish. |
| `D` Derived | Rebuildable projection, delivery state, or search/authorization helper | May be rebuilt from canonical sources. |
| `S` Sensitive canonical | Canonical data requiring additional field/resource controls | Same as canonical plus restricted read/audit policy. |

### 3.5 Module Ownership

- Core modules own reusable primitives and their tables.
- CRM, Sales, and HR own private business tables.
- A module can hold a foreign key to a public Core identity/resource contract.
- One business plugin MUST NOT foreign-key directly to another plugin's private table. Cross-plugin references use `core.resource` and public contracts/events.
- Core MUST NOT foreign-key to plugin-private tables.

### 3.6 Deletion and Retention

- Ordinary deletion is archive-first.
- Security/audit events, approval decisions, published workflow versions, document revisions, domain events, and outbox payload facts are append-only.
- Identity credentials and sessions may be revoked and later purged under security retention policy without deleting attributable audit facts.
- Hard deletion is allowed only for explicitly erasable data, failed/quarantined uploads, or retention jobs with policy authority.
- Parent deletion MUST NOT cascade into audit, event, approval decision, or material history loss.
- Legal retention durations and jurisdiction-specific erasure remain policy inputs for Gate 3 notes.

## 4. Cross-Domain Context Map

```mermaid
erDiagram
    TENANT ||--o{ MEMBERSHIP : contains
    USER ||--o{ MEMBERSHIP : joins
    MEMBERSHIP ||--o{ ROLE_ASSIGNMENT : receives
    ROLE ||--o{ ROLE_ASSIGNMENT : assigned
    ROLE ||--o{ ROLE_PERMISSION : contains
    PERMISSION ||--o{ ROLE_PERMISSION : grants
    TENANT ||--o{ RESOURCE : owns
    RESOURCE_TYPE ||--o{ RESOURCE : classifies
    RESOURCE ||--o{ COMMENT : discusses
    RESOURCE ||--o{ ATTACHMENT : has
    RESOURCE ||--o{ ACTIVITY : projects
    RESOURCE ||--o{ WORKFLOW_BINDING : binds
    RESOURCE ||--o{ AUDIT_EVENT : subjects
    RESOURCE ||--o{ RESOURCE_RELATIONSHIP : source
    PLUGIN_INSTALLATION ||--o{ CAPABILITY_GRANT : receives
    RESOURCE ||--o| CRM_AGGREGATE : registers
    RESOURCE ||--o| SALES_AGGREGATE : registers
    RESOURCE ||--o| HR_AGGREGATE : registers
```

`CRM_AGGREGATE`, `SALES_AGGREGATE`, and `HR_AGGREGATE` are diagram aliases representing plugin-owned aggregate roots; they are not physical entities.

## 5. Foundation and Identity ERD

```mermaid
erDiagram
    TENANT ||--o{ RESOURCE : owns
    RESOURCE_TYPE ||--o{ RESOURCE : classifies
    USER ||--o{ CREDENTIAL : authenticates
    USER ||--o{ SESSION : opens
    USER ||--o{ MFA_FACTOR : enrolls
    IDENTITY_PROVIDER ||--o{ FEDERATED_IDENTITY : issues
    USER ||--o{ FEDERATED_IDENTITY : links
    USER ||--o| USER_PROFILE : has
    TENANT ||--o{ SERVICE_PRINCIPAL : owns
    SERVICE_PRINCIPAL ||--o{ SERVICE_CREDENTIAL : authenticates

    TENANT {
      uuid id PK
      string key UK
      string name
      string status
      timestamp created_at
    }
    RESOURCE_TYPE {
      uuid id PK
      string key UK
      string owner_module
      string permission_namespace
      string status
    }
    RESOURCE {
      uuid id PK
      uuid tenant_id FK
      uuid resource_type_id FK
      uuid concrete_id
      uuid organization_id
      uuid business_unit_id
      uuid department_id
      uuid team_id
      uuid location_id
      uuid owner_membership_id
      string lifecycle_state
      bigint version
    }
    USER {
      uuid id PK
      string email UK
      string username UK
      string status
      timestamp created_at
    }
    CREDENTIAL {
      uuid id PK
      uuid user_id FK
      string credential_type
      string secret_hash
      string status
      timestamp rotated_at
    }
    SESSION {
      uuid id PK
      uuid user_id FK
      string token_hash UK
      timestamp expires_at
      timestamp revoked_at
    }
    MFA_FACTOR {
      uuid id PK
      uuid user_id FK
      string factor_type
      string status
      timestamp verified_at
    }
    IDENTITY_PROVIDER {
      uuid id PK
      uuid tenant_id FK
      string provider_type
      string issuer
      string status
    }
    FEDERATED_IDENTITY {
      uuid id PK
      uuid identity_provider_id FK
      uuid user_id FK
      string subject
    }
    USER_PROFILE {
      uuid id PK
      uuid user_id FK
      string display_name
      string locale
      string timezone
    }
    SERVICE_PRINCIPAL {
      uuid id PK
      uuid tenant_id FK
      string key
      string principal_type
      string status
    }
    SERVICE_CREDENTIAL {
      uuid id PK
      uuid service_principal_id FK
      string secret_hash
      timestamp expires_at
      timestamp revoked_at
    }
```

## 6. Organization ERD

```mermaid
erDiagram
    TENANT ||--o{ ORGANIZATION : contains
    ORGANIZATION ||--o{ BUSINESS_UNIT : contains
    ORGANIZATION ||--o{ DEPARTMENT : contains
    BUSINESS_UNIT o|--o{ DEPARTMENT : groups
    DEPARTMENT ||--o{ TEAM : contains
    ORGANIZATION ||--o{ POSITION : defines
    ORGANIZATION ||--o{ LOCATION : operates
    ORGANIZATION ||--o{ COST_CENTER : defines
    USER ||--o{ MEMBERSHIP : joins
    TENANT ||--o{ MEMBERSHIP : contains
    MEMBERSHIP ||--o{ MEMBERSHIP_ORG_PLACEMENT : placed
    ORGANIZATION ||--o{ MEMBERSHIP_ORG_PLACEMENT : includes
    DEPARTMENT o|--o{ MEMBERSHIP_ORG_PLACEMENT : places
    POSITION o|--o{ MEMBERSHIP_ORG_PLACEMENT : appoints
    MEMBERSHIP ||--o{ TEAM_MEMBERSHIP : joins
    TEAM ||--o{ TEAM_MEMBERSHIP : includes
    ORGANIZATION ||--o{ ORG_HIERARCHY_CLOSURE : ancestors

    ORGANIZATION {
      uuid id PK
      uuid tenant_id FK
      string key
      string name
      string status
    }
    BUSINESS_UNIT {
      uuid id PK
      uuid tenant_id FK
      uuid organization_id FK
      uuid parent_id FK
      string key
      string status
    }
    DEPARTMENT {
      uuid id PK
      uuid tenant_id FK
      uuid organization_id FK
      uuid business_unit_id FK
      uuid parent_id FK
      string key
      string status
    }
    TEAM {
      uuid id PK
      uuid tenant_id FK
      uuid department_id FK
      uuid parent_id FK
      string key
      string status
    }
    POSITION {
      uuid id PK
      uuid tenant_id FK
      uuid organization_id FK
      string key
      string title
      string status
    }
    LOCATION {
      uuid id PK
      uuid tenant_id FK
      uuid organization_id FK
      uuid parent_id FK
      string key
      string status
    }
    COST_CENTER {
      uuid id PK
      uuid tenant_id FK
      uuid organization_id FK
      uuid parent_id FK
      string code
      string status
    }
    MEMBERSHIP {
      uuid id PK
      uuid tenant_id FK
      uuid user_id FK
      string membership_type
      string status
      timestamp joined_at
      timestamp terminated_at
    }
    MEMBERSHIP_ORG_PLACEMENT {
      uuid id PK
      uuid tenant_id FK
      uuid membership_id FK
      uuid organization_id FK
      uuid business_unit_id FK
      uuid department_id FK
      uuid position_id FK
      uuid location_id FK
      date effective_from
      date effective_to
      boolean is_primary
    }
    TEAM_MEMBERSHIP {
      uuid id PK
      uuid tenant_id FK
      uuid membership_id FK
      uuid team_id FK
      string member_role
      date effective_from
      date effective_to
    }
    ORG_HIERARCHY_CLOSURE {
      uuid tenant_id FK
      string node_type
      uuid ancestor_id
      uuid descendant_id
      integer depth
    }
```

## 7. Authorization ERD

```mermaid
erDiagram
    PERMISSION ||--o{ ROLE_PERMISSION : included
    ROLE ||--o{ ROLE_PERMISSION : contains
    MEMBERSHIP ||--o{ ROLE_ASSIGNMENT : receives
    ROLE ||--o{ ROLE_ASSIGNMENT : assigns
    RESOURCE o|--o{ ROLE_ASSIGNMENT : record_scope
    SERVICE_PRINCIPAL ||--o{ SERVICE_ROLE_ASSIGNMENT : receives
    ROLE ||--o{ SERVICE_ROLE_ASSIGNMENT : assigns
    ROLE_ASSIGNMENT ||--o{ ROLE_ASSIGNMENT_EVENT : records

    PERMISSION {
      uuid id PK
      string key UK
      string owner_module
      string resource_key
      string action
      string sensitivity
      string status
    }
    ROLE {
      uuid id PK
      uuid tenant_id FK
      string key
      string name
      string role_type
      boolean is_reserved
      string status
      bigint version
    }
    ROLE_PERMISSION {
      uuid tenant_id FK
      uuid role_id FK
      uuid permission_id FK
      timestamp granted_at
      uuid granted_by FK
    }
    ROLE_ASSIGNMENT {
      uuid id PK
      uuid tenant_id FK
      uuid membership_id FK
      uuid role_id FK
      string scope_type
      uuid scope_anchor_id
      uuid resource_id FK
      string status
      timestamp effective_from
      timestamp expires_at
    }
    SERVICE_ROLE_ASSIGNMENT {
      uuid id PK
      uuid tenant_id FK
      uuid service_principal_id FK
      uuid role_id FK
      string scope_type
      uuid scope_anchor_id
      string status
    }
    ROLE_ASSIGNMENT_EVENT {
      uuid id PK
      uuid tenant_id FK
      uuid role_assignment_id FK
      string event_type
      uuid actor_membership_id FK
      string reason
      timestamp occurred_at
    }
```

The polymorphic organizational `scope_anchor_id` is constrained by `scope_type` and same-tenant scope-anchor validation. `record` scope uses the concrete `resource_id` FK. Gate 3 MUST implement an enforceable scope anchor strategy; free-form UUIDs are not acceptable.

## 8. Core Data and Search ERD

```mermaid
erDiagram
    OBJECT_TYPE ||--o{ OBJECT_TYPE_VERSION : versions
    OBJECT_TYPE_VERSION ||--o{ FIELD_DEFINITION : defines
    FIELD_DEFINITION ||--o{ FIELD_OPTION : offers
    OBJECT_TYPE ||--o{ RECORD : classifies
    RESOURCE ||--|| RECORD : registers
    RECORD ||--o{ RECORD_VALUE : contains
    FIELD_DEFINITION ||--o{ RECORD_VALUE : validates
    RELATIONSHIP_TYPE ||--o{ RESOURCE_RELATIONSHIP : types
    RESOURCE ||--o{ RESOURCE_RELATIONSHIP : source
    RESOURCE ||--o{ RESOURCE_RELATIONSHIP : target
    MEMBERSHIP ||--o{ SAVED_QUERY : owns

    OBJECT_TYPE {
      uuid id PK
      uuid tenant_id FK
      string key
      string owner_module
      string status
      integer current_version
    }
    OBJECT_TYPE_VERSION {
      uuid id PK
      uuid tenant_id FK
      uuid object_type_id FK
      integer version_number
      string status
      timestamp published_at
    }
    FIELD_DEFINITION {
      uuid id PK
      uuid tenant_id FK
      uuid object_type_version_id FK
      string key
      string data_type
      string classification
      boolean required
      json validation
    }
    FIELD_OPTION {
      uuid id PK
      uuid tenant_id FK
      uuid field_definition_id FK
      string value
      string label
      integer sort_order
    }
    RECORD {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid object_type_id FK
      string identifier
      string status
      bigint version
    }
    RECORD_VALUE {
      uuid id PK
      uuid tenant_id FK
      uuid record_id FK
      uuid field_definition_id FK
      json typed_value
    }
    RELATIONSHIP_TYPE {
      uuid id PK
      uuid tenant_id FK
      string key
      string source_type_key
      string target_type_key
      string cardinality
      string status
    }
    RESOURCE_RELATIONSHIP {
      uuid id PK
      uuid tenant_id FK
      uuid relationship_type_id FK
      uuid source_resource_id FK
      uuid target_resource_id FK
      string status
      json metadata
    }
    SAVED_QUERY {
      uuid id PK
      uuid tenant_id FK
      uuid owner_membership_id FK
      string target_resource_type
      json filter_definition
      string visibility
    }
```

`RECORD_VALUE.typed_value` is a logical discriminated value, not an unrestricted schema escape hatch. Gate 3 MUST choose typed physical storage and enforce exactly one value compatible with the published field definition.

## 9. Work ERD

```mermaid
erDiagram
    RESOURCE ||--|| TASK : registers
    RESOURCE ||--|| PROJECT : registers
    RESOURCE ||--|| MILESTONE : registers
    RESOURCE ||--|| REQUEST : registers
    PROJECT ||--o{ PROJECT_MEMBER : includes
    MEMBERSHIP ||--o{ PROJECT_MEMBER : participates
    PROJECT ||--o{ MILESTONE : contains
    PROJECT o|--o{ TASK : contains
    RESOURCE ||--o{ WORK_ASSIGNMENT : assigned
    MEMBERSHIP o|--o{ WORK_ASSIGNMENT : member
    TEAM o|--o{ WORK_ASSIGNMENT : team
    RESOURCE ||--o{ WORK_DEPENDENCY : predecessor
    RESOURCE ||--o{ WORK_DEPENDENCY : successor
    RESOURCE ||--o{ WORK_STATUS_HISTORY : changes

    TASK {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid project_id FK
      string title
      string priority
      string status
      date due_date
      uuid source_resource_id FK
      bigint version
    }
    PROJECT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string key
      string name
      string status
      date start_date
      date target_date
    }
    PROJECT_MEMBER {
      uuid id PK
      uuid tenant_id FK
      uuid project_id FK
      uuid membership_id FK
      string member_role
      date effective_from
      date effective_to
    }
    MILESTONE {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid project_id FK
      string name
      string status
      date due_date
    }
    REQUEST {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string request_type
      string title
      string status
      uuid requester_membership_id FK
    }
    WORK_ASSIGNMENT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid membership_id FK
      uuid team_id FK
      string assignment_type
      string status
      timestamp assigned_at
    }
    WORK_DEPENDENCY {
      uuid id PK
      uuid tenant_id FK
      uuid predecessor_resource_id FK
      uuid successor_resource_id FK
      string dependency_type
    }
    WORK_STATUS_HISTORY {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string from_status
      string to_status
      uuid actor_membership_id FK
      timestamp occurred_at
    }
```

## 10. Workflow and Approval ERD

```mermaid
erDiagram
    WORKFLOW_DEFINITION ||--o{ WORKFLOW_VERSION : versions
    WORKFLOW_VERSION ||--o{ WORKFLOW_STATE : defines
    WORKFLOW_VERSION ||--o{ WORKFLOW_TRANSITION : defines
    WORKFLOW_STATE ||--o{ WORKFLOW_TRANSITION : from_state
    WORKFLOW_STATE ||--o{ WORKFLOW_TRANSITION : to_state
    WORKFLOW_TRANSITION ||--o{ TRANSITION_CONDITION : guards
    WORKFLOW_TRANSITION ||--o{ TRANSITION_ACTION : executes
    WORKFLOW_VERSION ||--o{ WORKFLOW_BINDING : binds
    RESOURCE_TYPE ||--o{ WORKFLOW_BINDING : targets
    RESOURCE ||--o{ WORKFLOW_INSTANCE : governs
    WORKFLOW_VERSION ||--o{ WORKFLOW_INSTANCE : instantiates
    WORKFLOW_INSTANCE ||--o{ TRANSITION_EXECUTION : records
    WORKFLOW_TRANSITION ||--o{ TRANSITION_EXECUTION : executes
    WORKFLOW_INSTANCE ||--o{ APPROVAL : requests
    APPROVAL ||--o{ APPROVAL_STEP : sequences
    APPROVAL_STEP ||--o{ APPROVAL_ASSIGNEE : assigns
    APPROVAL_STEP ||--o{ APPROVAL_DECISION : decides
    APPROVAL_ASSIGNEE ||--o{ APPROVAL_DELEGATION : delegates

    WORKFLOW_DEFINITION {
      uuid id PK
      uuid tenant_id FK
      string key
      string owner_module
      string status
    }
    WORKFLOW_VERSION {
      uuid id PK
      uuid tenant_id FK
      uuid workflow_definition_id FK
      integer version_number
      string status
      timestamp published_at
    }
    WORKFLOW_STATE {
      uuid id PK
      uuid tenant_id FK
      uuid workflow_version_id FK
      string key
      string state_type
      boolean is_initial
      boolean is_terminal
    }
    WORKFLOW_TRANSITION {
      uuid id PK
      uuid tenant_id FK
      uuid workflow_version_id FK
      uuid from_state_id FK
      uuid to_state_id FK
      string key
      string required_permission_key
    }
    TRANSITION_CONDITION {
      uuid id PK
      uuid tenant_id FK
      uuid transition_id FK
      string condition_type
      json expression
      integer sort_order
    }
    TRANSITION_ACTION {
      uuid id PK
      uuid tenant_id FK
      uuid transition_id FK
      string action_type
      json configuration
      integer sort_order
    }
    WORKFLOW_BINDING {
      uuid id PK
      uuid tenant_id FK
      uuid workflow_version_id FK
      uuid resource_type_id FK
      string status
      timestamp effective_from
    }
    WORKFLOW_INSTANCE {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid workflow_version_id FK
      uuid current_state_id FK
      string status
      bigint version
    }
    TRANSITION_EXECUTION {
      uuid id PK
      uuid tenant_id FK
      uuid workflow_instance_id FK
      uuid transition_id FK
      uuid actor_membership_id FK
      string result
      string reason
      timestamp occurred_at
    }
    APPROVAL {
      uuid id PK
      uuid tenant_id FK
      uuid workflow_instance_id FK
      uuid resource_id FK
      string status
      timestamp requested_at
      timestamp completed_at
    }
    APPROVAL_STEP {
      uuid id PK
      uuid tenant_id FK
      uuid approval_id FK
      integer step_number
      string decision_rule
      string status
    }
    APPROVAL_ASSIGNEE {
      uuid id PK
      uuid tenant_id FK
      uuid approval_step_id FK
      uuid membership_id FK
      uuid role_id FK
      string status
    }
    APPROVAL_DECISION {
      uuid id PK
      uuid tenant_id FK
      uuid approval_step_id FK
      uuid actor_membership_id FK
      uuid original_assignee_id FK
      string decision
      string reason
      timestamp decided_at
    }
    APPROVAL_DELEGATION {
      uuid id PK
      uuid tenant_id FK
      uuid approval_assignee_id FK
      uuid from_membership_id FK
      uuid to_membership_id FK
      timestamp effective_from
      timestamp expires_at
      string status
    }
```

Published workflow definitions and all child states, transitions, conditions, and actions are immutable. Runtime instances always reference one published version.

## 11. Content ERD

```mermaid
erDiagram
    RESOURCE ||--|| DOCUMENT : registers
    DOCUMENT ||--o{ DOCUMENT_VERSION : versions
    FILE ||--o{ FILE_SCAN : scans
    FILE ||--o{ DOCUMENT_VERSION : stores
    FOLDER ||--o{ FOLDER : contains
    FOLDER ||--o{ DOCUMENT : contains
    FOLDER ||--o{ FOLDER_CLOSURE : ancestors
    FILE ||--o{ ATTACHMENT : attaches
    DOCUMENT o|--o{ ATTACHMENT : attaches
    RESOURCE ||--o{ ATTACHMENT : target
    UPLOAD_SESSION ||--o| FILE : produces

    FILE {
      uuid id PK
      uuid tenant_id FK
      string object_key UK
      string original_name
      string media_type
      bigint size_bytes
      string checksum
      string status
      string classification
    }
    FILE_SCAN {
      uuid id PK
      uuid tenant_id FK
      uuid file_id FK
      string scanner
      string result
      timestamp scanned_at
    }
    DOCUMENT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid folder_id FK
      string key
      string title
      string classification
      string status
      integer current_version
    }
    DOCUMENT_VERSION {
      uuid id PK
      uuid tenant_id FK
      uuid document_id FK
      uuid file_id FK
      integer version_number
      string title
      timestamp created_at
    }
    FOLDER {
      uuid id PK
      uuid tenant_id FK
      uuid parent_id FK
      string name
      string classification
      string status
    }
    FOLDER_CLOSURE {
      uuid tenant_id FK
      uuid ancestor_id FK
      uuid descendant_id FK
      integer depth
    }
    ATTACHMENT {
      uuid id PK
      uuid tenant_id FK
      uuid target_resource_id FK
      uuid file_id FK
      uuid document_id FK
      string status
    }
    UPLOAD_SESSION {
      uuid id PK
      uuid tenant_id FK
      uuid uploader_membership_id FK
      string object_key UK
      string status
      timestamp expires_at
    }
```

An attachment references exactly one of `file_id` or `document_id`. Binary data remains in S3-compatible object storage; PostgreSQL stores metadata and integrity references only.

## 12. Communication ERD

```mermaid
erDiagram
    RESOURCE ||--o{ COMMENT : discusses
    MEMBERSHIP ||--o{ COMMENT : authors
    COMMENT ||--o{ COMMENT_REVISION : revisions
    COMMENT ||--o{ MENTION : contains
    MEMBERSHIP ||--o{ MENTION : mentions
    MEMBERSHIP ||--o{ NOTIFICATION : receives
    NOTIFICATION ||--o{ NOTIFICATION_DELIVERY : delivers
    MEMBERSHIP ||--o{ SUBSCRIPTION : subscribes
    RESOURCE o|--o{ SUBSCRIPTION : follows

    COMMENT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid author_membership_id FK
      string body
      string status
      bigint version
    }
    COMMENT_REVISION {
      uuid id PK
      uuid tenant_id FK
      uuid comment_id FK
      integer revision_number
      string body
      uuid editor_membership_id FK
      timestamp created_at
    }
    MENTION {
      uuid id PK
      uuid tenant_id FK
      uuid comment_id FK
      uuid mentioned_membership_id FK
      integer start_offset
      integer end_offset
    }
    NOTIFICATION {
      uuid id PK
      uuid tenant_id FK
      uuid recipient_membership_id FK
      uuid resource_id FK
      string notification_type
      json safe_payload
      timestamp read_at
    }
    NOTIFICATION_DELIVERY {
      uuid id PK
      uuid tenant_id FK
      uuid notification_id FK
      string channel
      string status
      integer attempt_count
      timestamp delivered_at
    }
    SUBSCRIPTION {
      uuid id PK
      uuid tenant_id FK
      uuid membership_id FK
      uuid resource_id FK
      string event_category
      string status
    }
```

## 13. Automation ERD

```mermaid
erDiagram
    AUTOMATION_RULE ||--o{ AUTOMATION_VERSION : versions
    AUTOMATION_VERSION ||--o{ AUTOMATION_TRIGGER : triggers
    AUTOMATION_VERSION ||--o{ AUTOMATION_CONDITION : guards
    AUTOMATION_VERSION ||--o{ AUTOMATION_ACTION : executes
    AUTOMATION_RULE ||--o{ AUTOMATION_EXECUTION : runs
    AUTOMATION_VERSION ||--o{ AUTOMATION_EXECUTION : uses
    AUTOMATION_EXECUTION ||--o{ AUTOMATION_ACTION_RESULT : results
    IDEMPOTENCY_KEY o|--o{ AUTOMATION_EXECUTION : deduplicates

    AUTOMATION_RULE {
      uuid id PK
      uuid tenant_id FK
      string key
      string owner_module
      string status
      integer current_version
    }
    AUTOMATION_VERSION {
      uuid id PK
      uuid tenant_id FK
      uuid automation_rule_id FK
      integer version_number
      string status
      timestamp published_at
    }
    AUTOMATION_TRIGGER {
      uuid id PK
      uuid tenant_id FK
      uuid automation_version_id FK
      string trigger_type
      json configuration
    }
    AUTOMATION_CONDITION {
      uuid id PK
      uuid tenant_id FK
      uuid automation_version_id FK
      string condition_type
      json expression
      integer sort_order
    }
    AUTOMATION_ACTION {
      uuid id PK
      uuid tenant_id FK
      uuid automation_version_id FK
      string action_type
      json configuration
      integer sort_order
    }
    AUTOMATION_EXECUTION {
      uuid id PK
      uuid tenant_id FK
      uuid automation_rule_id FK
      uuid automation_version_id FK
      uuid trigger_resource_id FK
      string status
      timestamp started_at
      timestamp completed_at
    }
    AUTOMATION_ACTION_RESULT {
      uuid id PK
      uuid tenant_id FK
      uuid automation_execution_id FK
      uuid automation_action_id FK
      string status
      json safe_result
      integer attempt_count
    }
```

Published automation versions and their trigger/condition/action children are immutable. Secrets in configuration are stored as references, never embedded values.

## 14. Activity, Audit, Event, and Reliability ERD

```mermaid
erDiagram
    RESOURCE o|--o{ ACTIVITY : projects
    RESOURCE o|--o{ AUDIT_EVENT : subjects
    RESOURCE o|--o{ SECURITY_EVENT : subjects
    DOMAIN_EVENT ||--o| OUTBOX_MESSAGE : publishes
    EVENT_SUBSCRIPTION ||--o{ EVENT_DELIVERY : receives
    DOMAIN_EVENT ||--o{ EVENT_DELIVERY : delivers
    IDEMPOTENCY_KEY o|--o{ EVENT_DELIVERY : deduplicates

    ACTIVITY {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string activity_type
      uuid actor_membership_id FK
      json safe_payload
      timestamp occurred_at
    }
    AUDIT_EVENT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string action
      string permission_key
      uuid actor_principal_id
      string result
      json safe_before
      json safe_after
      timestamp occurred_at
    }
    SECURITY_EVENT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string event_type
      string severity
      uuid actor_principal_id
      json safe_context
      timestamp occurred_at
    }
    DOMAIN_EVENT {
      uuid id PK
      uuid tenant_id FK
      string event_type
      integer schema_version
      uuid resource_id FK
      uuid actor_principal_id
      uuid correlation_id
      uuid causation_id
      json payload
      timestamp occurred_at
    }
    OUTBOX_MESSAGE {
      uuid id PK
      uuid tenant_id FK
      uuid domain_event_id FK
      string status
      integer attempt_count
      timestamp available_at
      timestamp published_at
    }
    EVENT_SUBSCRIPTION {
      uuid id PK
      string owner_module
      string subscriber_key
      string event_type
      integer min_version
      integer max_version
      string status
    }
    EVENT_DELIVERY {
      uuid id PK
      uuid tenant_id FK
      uuid domain_event_id FK
      uuid event_subscription_id FK
      string status
      integer attempt_count
      timestamp next_attempt_at
    }
    IDEMPOTENCY_KEY {
      uuid id PK
      uuid tenant_id FK
      string namespace
      string key
      string request_hash
      uuid result_resource_id FK
      timestamp expires_at
    }
```

Audit, security, domain event, and activity facts are insert-only. Outbox/delivery operational status is mutable, but its event identity and payload reference are immutable.

## 15. Plugin, Configuration, and Integration ERD

```mermaid
erDiagram
    PLUGIN ||--o{ PLUGIN_VERSION : versions
    PLUGIN_VERSION ||--o{ PLUGIN_DEPENDENCY : declares
    PLUGIN_VERSION ||--o{ CAPABILITY_DEFINITION : declares
    PLUGIN ||--o{ PLUGIN_INSTALLATION : installed
    TENANT ||--o{ PLUGIN_INSTALLATION : enables
    PLUGIN_INSTALLATION ||--o{ CAPABILITY_GRANT : receives
    CAPABILITY_DEFINITION ||--o{ CAPABILITY_GRANT : grants
    PLUGIN_INSTALLATION ||--o{ PLUGIN_CONFIGURATION : configures
    PLUGIN_VERSION ||--o{ PLUGIN_MIGRATION : contains
    PLUGIN_INSTALLATION ||--o{ PLUGIN_LIFECYCLE_EVENT : records
    TENANT ||--o{ TENANT_SETTING : configures
    TENANT ||--o{ FEATURE_SETTING : configures
    TENANT ||--o{ INTEGRATION_CONNECTION : owns
    INTEGRATION_CONNECTION ||--o{ WEBHOOK : exposes
    WEBHOOK ||--o{ WEBHOOK_DELIVERY : delivers

    PLUGIN {
      uuid id PK
      string key UK
      string name
      string status
    }
    PLUGIN_VERSION {
      uuid id PK
      uuid plugin_id FK
      string version
      string manifest_digest
      string signature_status
      string status
    }
    PLUGIN_DEPENDENCY {
      uuid id PK
      uuid plugin_version_id FK
      string dependency_key
      string version_constraint
      boolean required
    }
    CAPABILITY_DEFINITION {
      uuid id PK
      uuid plugin_version_id FK
      string key
      string target_module
      string status
    }
    PLUGIN_INSTALLATION {
      uuid id PK
      uuid tenant_id FK
      uuid plugin_id FK
      uuid plugin_version_id FK
      string status
      timestamp installed_at
    }
    CAPABILITY_GRANT {
      uuid id PK
      uuid tenant_id FK
      uuid plugin_installation_id FK
      uuid capability_definition_id FK
      string status
      timestamp granted_at
    }
    PLUGIN_CONFIGURATION {
      uuid id PK
      uuid tenant_id FK
      uuid plugin_installation_id FK
      string key
      json non_secret_value
      string secret_reference
    }
    PLUGIN_MIGRATION {
      uuid id PK
      uuid plugin_version_id FK
      string migration_key
      string checksum
      integer sequence
    }
    PLUGIN_LIFECYCLE_EVENT {
      uuid id PK
      uuid tenant_id FK
      uuid plugin_installation_id FK
      string from_status
      string to_status
      uuid actor_membership_id FK
      timestamp occurred_at
    }
    TENANT_SETTING {
      uuid id PK
      uuid tenant_id FK
      string key
      json non_secret_value
      string secret_reference
      bigint version
    }
    FEATURE_SETTING {
      uuid id PK
      uuid tenant_id FK
      string feature_key
      boolean enabled
      json configuration
    }
    INTEGRATION_CONNECTION {
      uuid id PK
      uuid tenant_id FK
      string key
      string connection_type
      json non_secret_config
      string secret_reference
      string status
    }
    WEBHOOK {
      uuid id PK
      uuid tenant_id FK
      uuid integration_connection_id FK
      string endpoint
      string secret_reference
      string status
    }
    WEBHOOK_DELIVERY {
      uuid id PK
      uuid tenant_id FK
      uuid webhook_id FK
      uuid domain_event_id FK
      string status
      integer attempt_count
      timestamp next_attempt_at
    }
```

Catalog entities through `PLUGIN_MIGRATION` are platform-scoped unless explicitly tied to `PLUGIN_INSTALLATION`. Tenant configuration and secret references never contain raw secrets.

## 16. CRM ERD

```mermaid
erDiagram
    RESOURCE ||--|| CRM_ACCOUNT : registers
    RESOURCE ||--|| CRM_CONTACT : registers
    RESOURCE ||--|| CRM_LEAD : registers
    RESOURCE ||--|| CRM_OPPORTUNITY : registers
    CRM_ACCOUNT ||--o{ CRM_ACCOUNT_CONTACT : has
    CRM_CONTACT ||--o{ CRM_ACCOUNT_CONTACT : belongs
    CRM_PIPELINE ||--o{ CRM_PIPELINE_STAGE : defines
    CRM_PIPELINE_STAGE ||--o{ CRM_OPPORTUNITY : stages
    CRM_ACCOUNT o|--o{ CRM_OPPORTUNITY : concerns
    CRM_LEAD o|--o{ CRM_LEAD_CONVERSION : converts
    RESOURCE ||--o{ CRM_ACTIVITY_LINK : relates

    CRM_ACCOUNT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string account_number
      string name
      string status
      uuid owner_membership_id FK
    }
    CRM_CONTACT {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string display_name
      string primary_email
      string status
      uuid owner_membership_id FK
    }
    CRM_ACCOUNT_CONTACT {
      uuid id PK
      uuid tenant_id FK
      uuid account_id FK
      uuid contact_id FK
      string relationship_role
      boolean is_primary
    }
    CRM_LEAD {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string display_name
      string source
      string status
      uuid owner_membership_id FK
    }
    CRM_LEAD_CONVERSION {
      uuid id PK
      uuid tenant_id FK
      uuid lead_id FK
      uuid account_resource_id FK
      uuid contact_resource_id FK
      uuid opportunity_resource_id FK
      uuid actor_membership_id FK
      timestamp converted_at
    }
    CRM_PIPELINE {
      uuid id PK
      uuid tenant_id FK
      string key
      string name
      string status
      integer current_version
    }
    CRM_PIPELINE_STAGE {
      uuid id PK
      uuid tenant_id FK
      uuid pipeline_id FK
      string key
      string name
      integer sequence
      decimal probability
    }
    CRM_OPPORTUNITY {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid pipeline_stage_id FK
      uuid account_id FK
      string name
      decimal value
      string currency
      string status
      uuid owner_membership_id FK
    }
    CRM_ACTIVITY_LINK {
      uuid id PK
      uuid tenant_id FK
      uuid crm_resource_id FK
      uuid activity_resource_id FK
      string activity_kind
    }
```

Lead conversion targets are `core.resource` references so CRM does not directly depend on Sales-private tables. The conversion transaction validates target resource types.

## 17. Sales ERD

```mermaid
erDiagram
    RESOURCE ||--|| QUOTATION : registers
    QUOTATION ||--o{ QUOTATION_LINE : contains
    RESOURCE ||--|| SALES_ORDER : registers
    SALES_ORDER ||--o{ SALES_ORDER_LINE : contains
    QUOTATION o|--o{ SALES_ORDER : originates
    PRICING_RULE ||--o{ PRICING_RULE_VERSION : versions
    PRICING_RULE_VERSION ||--o{ PRICE_ADJUSTMENT : defines
    QUOTATION ||--o{ DISCOUNT_REQUEST : requests
    SALES_ORDER ||--o{ DISCOUNT_REQUEST : requests
    DISCOUNT_REQUEST ||--o{ DISCOUNT_DECISION : decides

    QUOTATION {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string quotation_number
      uuid customer_resource_id FK
      string status
      string currency
      decimal subtotal
      decimal total
      timestamp valid_until
    }
    QUOTATION_LINE {
      uuid id PK
      uuid tenant_id FK
      uuid quotation_id FK
      integer line_number
      uuid item_resource_id FK
      string description
      decimal quantity
      decimal unit_price
      decimal line_total
    }
    SALES_ORDER {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid quotation_id FK
      string order_number
      uuid customer_resource_id FK
      string status
      string currency
      decimal total
      timestamp approved_at
    }
    SALES_ORDER_LINE {
      uuid id PK
      uuid tenant_id FK
      uuid sales_order_id FK
      integer line_number
      uuid item_resource_id FK
      string description
      decimal quantity
      decimal unit_price
      decimal line_total
    }
    PRICING_RULE {
      uuid id PK
      uuid tenant_id FK
      string key
      string name
      string status
      integer current_version
    }
    PRICING_RULE_VERSION {
      uuid id PK
      uuid tenant_id FK
      uuid pricing_rule_id FK
      integer version_number
      string status
      timestamp published_at
    }
    PRICE_ADJUSTMENT {
      uuid id PK
      uuid tenant_id FK
      uuid pricing_rule_version_id FK
      string adjustment_type
      decimal value
      json condition
      integer priority
    }
    DISCOUNT_REQUEST {
      uuid id PK
      uuid tenant_id FK
      uuid quotation_id FK
      uuid sales_order_id FK
      uuid requester_membership_id FK
      decimal requested_value
      string reason
      string status
    }
    DISCOUNT_DECISION {
      uuid id PK
      uuid tenant_id FK
      uuid discount_request_id FK
      uuid actor_membership_id FK
      string decision
      string reason
      timestamp decided_at
    }
```

A discount request references exactly one quotation or sales order. Customer and item links use registered resources and public module contracts; Sales does not depend on CRM or Inventory private tables.

## 18. HR ERD

```mermaid
erDiagram
    RESOURCE ||--|| EMPLOYEE : registers
    MEMBERSHIP o|--o| EMPLOYEE : represents
    EMPLOYEE ||--o{ EMPLOYMENT : has
    EMPLOYMENT ||--o{ COMPENSATION : compensated
    EMPLOYEE ||--o{ LEAVE_REQUEST : requests
    RESOURCE ||--|| LEAVE_REQUEST : registers
    RESOURCE ||--|| REQUISITION : registers
    REQUISITION ||--o{ APPLICATION : receives
    CANDIDATE ||--o{ APPLICATION : submits
    APPLICATION ||--o{ INTERVIEW : schedules
    RESOURCE ||--|| CANDIDATE : registers
    RESOURCE ||--|| APPLICATION : registers
    EMPLOYEE ||--o{ ONBOARDING_INSTANCE : participates
    ONBOARDING_PLAN ||--o{ ONBOARDING_PLAN_STEP : defines
    ONBOARDING_PLAN ||--o{ ONBOARDING_INSTANCE : instantiates
    ONBOARDING_INSTANCE ||--o{ ONBOARDING_STEP : executes
    ONBOARDING_PLAN_STEP ||--o{ ONBOARDING_STEP : materializes

    EMPLOYEE {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid membership_id FK
      string employee_number
      string display_name
      string status
      date hire_date
    }
    EMPLOYMENT {
      uuid id PK
      uuid tenant_id FK
      uuid employee_id FK
      uuid organization_id FK
      uuid department_id FK
      uuid position_id FK
      uuid manager_employee_id FK
      string employment_type
      date effective_from
      date effective_to
      string status
    }
    COMPENSATION {
      uuid id PK
      uuid tenant_id FK
      uuid employment_id FK
      string compensation_type
      decimal amount
      string currency
      string frequency
      date effective_from
      date effective_to
    }
    LEAVE_REQUEST {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid employee_id FK
      string leave_type
      date start_date
      date end_date
      string status
    }
    REQUISITION {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string requisition_number
      uuid department_id FK
      uuid hiring_manager_membership_id FK
      string status
    }
    CANDIDATE {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      string display_name
      string email
      string consent_status
      string status
      timestamp retention_until
    }
    APPLICATION {
      uuid id PK
      uuid tenant_id FK
      uuid resource_id FK
      uuid requisition_id FK
      uuid candidate_id FK
      string stage
      string status
    }
    INTERVIEW {
      uuid id PK
      uuid tenant_id FK
      uuid application_id FK
      timestamp scheduled_at
      string interview_type
      string status
    }
    INTERVIEW_PARTICIPANT {
      uuid id PK
      uuid tenant_id FK
      uuid interview_id FK
      uuid membership_id FK
      string participant_role
    }
    INTERVIEW_FEEDBACK {
      uuid id PK
      uuid tenant_id FK
      uuid interview_id FK
      uuid author_membership_id FK
      json rating
      string notes
      timestamp submitted_at
    }
    ONBOARDING_PLAN {
      uuid id PK
      uuid tenant_id FK
      string key
      string name
      string status
      integer version_number
    }
    ONBOARDING_PLAN_STEP {
      uuid id PK
      uuid tenant_id FK
      uuid onboarding_plan_id FK
      string key
      string title
      integer sequence
      json configuration
    }
    ONBOARDING_INSTANCE {
      uuid id PK
      uuid tenant_id FK
      uuid onboarding_plan_id FK
      uuid employee_id FK
      string status
      date start_date
      date target_date
    }
    ONBOARDING_STEP {
      uuid id PK
      uuid tenant_id FK
      uuid onboarding_instance_id FK
      uuid onboarding_plan_step_id FK
      uuid work_resource_id FK
      string status
      timestamp completed_at
    }
```

Compensation is a separate restricted aggregate. Tenant Administrator receives no implicit read path. Interview feedback visibility is independent from candidate/application base visibility.

## 19. Entity Register

The register is authoritative for ownership, classification, tenancy, lifecycle, and retention. Diagram aliases are excluded.

### 19.1 Foundation, Identity, Organization, Authorization

| Entity | Owner | Class | Tenant rule | Alternate key / lifecycle | Retention |
|---|---|---|---|---|---|
| `tenant` | Core/Foundation | C | platform root | `key`; provisioning → active → suspended → closed | Retain business/audit anchor. |
| `resource_type` | Core/Foundation | C | platform | `key`; registered → active → retired | Never reuse key. |
| `resource` | Core/Foundation | C/D projection | tenant | type + concrete ID; active → archived | Retain while referenced. |
| `user` | Identity | S | platform identity | normalized email/username; invited → active → suspended → closed | Privacy/security policy. |
| `credential` | Identity | S | platform identity | type per user; active → rotated/revoked | Purge secret after retention. |
| `session` | Identity | S | platform identity + tenant context | token hash; active → expired/revoked | Short security retention. |
| `mfa_factor` | Identity | S | platform identity | factor key; pending → active → revoked | Purge secrets after revocation. |
| `identity_provider` | Identity | S | tenant | issuer/client key; active → disabled | Retain config audit. |
| `federated_identity` | Identity | S | tenant/provider-bound | provider + subject; active → unlinked | Retain link history safely. |
| `user_profile` | Identity | C | platform identity | one per user | Erasable subject to audit. |
| `service_principal` | Identity | S | tenant | tenant + key; active → disabled | Retain identity audit. |
| `service_credential` | Identity | S | parent-bound | credential fingerprint; active → expired/revoked | Purge secret material. |
| `organization` | Organization | C | tenant | tenant + key; active → archived | Archive; retain references. |
| `business_unit` | Organization | C | tenant | organization + key; active → archived | Archive; retain hierarchy. |
| `department` | Organization | C | tenant | organization + key; active → archived | Archive; retain hierarchy. |
| `team` | Organization | C | tenant | department + key; active → archived | Archive; retain history. |
| `position` | Organization | C | tenant | organization + key; active → archived | Archive. |
| `location` | Organization | C | tenant | organization + key; active → archived | Archive. |
| `cost_center` | Organization | C | tenant | organization + code; active → archived | Archive. |
| `membership` | Organization | C | tenant | tenant + user; invited → active → suspended → terminated | Retain attribution. |
| `membership_org_placement` | Organization | I/effective | tenant | member + interval + primary | Retain employment/org history. |
| `team_membership` | Organization | I/effective | tenant | member + team + interval | Retain access history. |
| `org_hierarchy_closure` | Organization | D | tenant | node type + ancestor + descendant | Rebuildable. |
| `permission` | Authorization | C | platform registry | permission key; active → retired | Never reuse key. |
| `role` | Authorization | C | tenant | tenant + key; active → archived | Retain assignment references. |
| `role_permission` | Authorization | C | tenant | role + permission | Retain grant audit separately. |
| `role_assignment` | Authorization | C | tenant | member + role + scope + interval; pending → active → suspended/expired/revoked | Retain permanently for access audit. |
| `service_role_assignment` | Authorization | C | tenant | service + role + scope; active → revoked | Retain permanently for access audit. |
| `role_assignment_event` | Authorization | I | tenant | event ID | Append-only. |

### 19.2 Core Data, Work, Workflow

| Entity | Owner | Class | Tenant rule | Alternate key / lifecycle | Retention |
|---|---|---|---|---|---|
| `object_type` | Core Data | C | tenant | tenant + owner + key; draft → active → archived | Retain schema identity. |
| `object_type_version` | Core Data | I | tenant | object type + version; draft → published/retired | Published immutable. |
| `field_definition` | Core Data | I when published | parent-bound | version + key | Retain with object version. |
| `field_option` | Core Data | I when published | parent-bound | field + value | Retain with field. |
| `record` | Core Data | C | tenant | object type + identifier; active → archived/deleted-by-policy | Policy-based. |
| `record_value` | Core Data | C | tenant | record + field | Follows record retention. |
| `relationship_type` | Core Data | C | tenant | tenant + key; active → archived | Retain relationship semantics. |
| `resource_relationship` | Core Data | C | tenant | type + endpoints; active → archived | Retain material links. |
| `saved_query` | Search | C | tenant | owner + name | Delete with owner policy. |
| `task` | Work | C | tenant | optional task key; state machine | Archive; retain material work. |
| `project` | Work | C | tenant | tenant + key; planned → active → completed/archived | Retain project history. |
| `project_member` | Work | I/effective | tenant | project + member + interval | Retain participation history. |
| `milestone` | Work | C | tenant | project + name/key; planned → reached/cancelled | Retain. |
| `request` | Work | C | tenant | request number; workflow lifecycle | Retain by request policy. |
| `work_assignment` | Work | C/history | tenant | resource + assignee + type + interval | Retain attribution. |
| `work_dependency` | Work | C | tenant | predecessor + successor + type | Archive with work. |
| `work_status_history` | Work | I | tenant | history ID | Append-only. |
| `workflow_definition` | Workflow | C | tenant | tenant + owner + key; draft → active → archived | Retain identity. |
| `workflow_version` | Workflow | I when published | tenant | definition + version; draft → published/retired | Published immutable. |
| `workflow_state` | Workflow | I when published | parent-bound | version + key | Retain with version. |
| `workflow_transition` | Workflow | I when published | parent-bound | version + key | Retain with version. |
| `transition_condition` | Workflow | I when published | parent-bound | transition + order | Retain with version. |
| `transition_action` | Workflow | I when published | parent-bound | transition + order | Retain with version. |
| `workflow_binding` | Workflow | C/effective | tenant | resource type + effective interval | Retain binding history. |
| `workflow_instance` | Workflow | C | tenant | resource + workflow instance | Retain lifecycle state. |
| `transition_execution` | Workflow | I | tenant | execution ID | Append-only. |
| `approval` | Workflow | C | tenant | workflow instance + request | Retain decision process. |
| `approval_step` | Workflow | C | parent-bound | approval + step number | Retain. |
| `approval_assignee` | Workflow | C/history | tenant | step + assignee | Retain attribution. |
| `approval_decision` | Workflow | I/S | tenant | decision ID | Append-only. |
| `approval_delegation` | Workflow | I/effective | tenant | assignee + interval | Append-only lifecycle events. |

### 19.3 Content, Communication, Automation, Observability

| Entity | Owner | Class | Tenant rule | Alternate key / lifecycle | Retention |
|---|---|---|---|---|---|
| `file` | Content | S | tenant | object key/checksum; quarantined → available → archived/deleted | Object/data retention policy. |
| `file_scan` | Content | I | tenant | file + scanner + scan time | Security retention. |
| `document` | Content | S | tenant | tenant + key; draft → active → archived | Business retention. |
| `document_version` | Content | I/S | tenant | document + version | Immutable; business retention. |
| `folder` | Content | C | tenant | parent + normalized name; active → archived | Archive. |
| `folder_closure` | Content | D | tenant | ancestor + descendant | Rebuildable. |
| `attachment` | Content | C | tenant | target + source | Retain association/history. |
| `upload_session` | Content | S | tenant | object key; initiated → uploaded → validated/expired | Short retention if incomplete. |
| `comment` | Communication | C | tenant | comment ID; active → archived | Parent policy. |
| `comment_revision` | Communication | I | tenant | comment + revision | Immutable. |
| `mention` | Communication | C | tenant | comment + member + offsets | Follows comment. |
| `notification` | Communication | D | tenant | recipient + event reference | Configurable short retention. |
| `notification_delivery` | Communication | D | tenant | notification + channel | Operational retention. |
| `subscription` | Communication | C | tenant | member + resource/category; active → archived | Delete/archive with member. |
| `automation_rule` | Automation | C | tenant | tenant + key; draft → enabled/disabled/archived | Retain identity. |
| `automation_version` | Automation | I when published | tenant | rule + version | Published immutable. |
| `automation_trigger` | Automation | I when published | parent-bound | version + trigger key | Retain with version. |
| `automation_condition` | Automation | I when published | parent-bound | version + order | Retain with version. |
| `automation_action` | Automation | I when published | parent-bound | version + order | Retain with version. |
| `automation_execution` | Automation | C/history | tenant | execution ID; queued → running → succeeded/failed/cancelled | Operational/audit retention. |
| `automation_action_result` | Automation | C/history | tenant | execution + action | Operational retention. |
| `activity` | Activity | I/D | tenant | activity ID | Append-only; may be reconstructable. |
| `audit_event` | Audit | I/S | tenant | event ID | Append-only security/business retention. |
| `security_event` | Audit | I/S | tenant or platform | event ID | Append-only security retention. |
| `domain_event` | Event | I | tenant | event ID | Append-only according to event retention. |
| `outbox_message` | Event | C operational + immutable payload ref | tenant | domain event; pending → published/dead | Retain delivery evidence. |
| `event_subscription` | Event | C | platform/module | subscriber + event type + version range | Retain compatibility history. |
| `event_delivery` | Event | C operational | tenant | event + subscription; pending → delivered/dead | Operational retention. |
| `idempotency_key` | Reliability | C/D | tenant | namespace + key | Bounded expiry after replay window. |

### 19.4 Plugin, Configuration, Integration

| Entity | Owner | Class | Tenant rule | Alternate key / lifecycle | Retention |
|---|---|---|---|---|---|
| `plugin` | Plugin | C | platform | plugin key; available → deprecated | Retain catalog identity. |
| `plugin_version` | Plugin | I | platform | plugin + semantic version | Immutable manifest. |
| `plugin_dependency` | Plugin | I | parent-bound | version + dependency | Retain with version. |
| `capability_definition` | Plugin | I | parent-bound | version + key | Retain with version. |
| `plugin_installation` | Plugin | C | tenant | tenant + plugin; installed → enabled → active → disabled/uninstalled | Retain lifecycle history. |
| `capability_grant` | Plugin | C/history | tenant | installation + capability; active → revoked | Retain security history. |
| `plugin_configuration` | Plugin | S | tenant | installation + key | Retain non-secret history as policy requires. |
| `plugin_migration` | Plugin | I | platform | plugin version + migration key | Permanent checksum record. |
| `plugin_lifecycle_event` | Plugin | I | tenant | event ID | Append-only. |
| `tenant_setting` | Configuration | S | tenant | tenant + key | Version/audit history. |
| `feature_setting` | Configuration | C | tenant | tenant + feature key | Version/audit history. |
| `integration_connection` | Integration | S | tenant | tenant + key; active → disabled/archived | Retain metadata; purge secret. |
| `webhook` | Integration | S | tenant | connection + endpoint key; active → disabled | Retain config audit. |
| `webhook_delivery` | Integration | C operational | tenant | webhook + event | Operational retention. |

### 19.5 CRM, Sales, HR

| Entity | Owner | Class | Tenant rule | Alternate key / lifecycle | Retention |
|---|---|---|---|---|---|
| `crm_account` | CRM | C | tenant | account number; active → archived | CRM policy. |
| `crm_contact` | CRM | S | tenant | optional external key; active → archived/erased-by-policy | Personal-data policy. |
| `crm_account_contact` | CRM | C | tenant | account + contact + role | Follows endpoints/history. |
| `crm_lead` | CRM | S | tenant | lead key; new → qualified/disqualified/converted | CRM/privacy policy. |
| `crm_lead_conversion` | CRM | I | tenant | lead (one successful conversion) | Permanent material history. |
| `crm_pipeline` | CRM | C | tenant | tenant + key; draft → published/archived | Retain versions/stages used. |
| `crm_pipeline_stage` | CRM | I when published | parent-bound | pipeline + key | Retain referenced stages. |
| `crm_opportunity` | CRM | C | tenant | opportunity key; staged → won/lost/archived | CRM policy. |
| `crm_activity_link` | CRM | C | tenant | CRM resource + activity resource | Retain with activity. |
| `quotation` | Sales | C | tenant | quotation number; draft → issued → accepted/rejected/expired/archived | Commercial retention. |
| `quotation_line` | Sales | C | parent-bound + tenant | quotation + line number | Follows quotation. |
| `sales_order` | Sales | C | tenant | order number; draft → approved → processing → completed/cancelled | Commercial/legal retention. |
| `sales_order_line` | Sales | C | parent-bound + tenant | order + line number | Follows order. |
| `pricing_rule` | Sales | C | tenant | tenant + key; draft → active/archived | Retain identity. |
| `pricing_rule_version` | Sales | I when published | tenant | rule + version | Published immutable. |
| `price_adjustment` | Sales | I when published | parent-bound | version + priority/key | Retain with version. |
| `discount_request` | Sales | S | tenant | request ID; pending → approved/rejected/cancelled | Commercial audit retention. |
| `discount_decision` | Sales | I/S | tenant | decision ID | Append-only. |
| `employee` | HR | S | tenant | employee number; pending → active → inactive/archived | Employment/privacy policy. |
| `employment` | HR | S/effective | tenant | employee + interval | Long-term employment record. |
| `compensation` | HR | S/effective | tenant | employment + type + interval | Restricted statutory retention. |
| `leave_request` | HR | S | tenant | request ID; draft → submitted → approved/rejected/cancelled | HR retention. |
| `requisition` | HR | S | tenant | requisition number; draft → open → filled/closed | Recruitment retention. |
| `candidate` | HR | S | tenant | candidate ID; active → archived/erased | Consent/retention deadline. |
| `application` | HR | S | tenant | requisition + candidate; staged → hired/rejected/withdrawn | Recruitment retention. |
| `interview` | HR | S | tenant | application + schedule | Recruitment retention. |
| `interview_participant` | HR | S | tenant | interview + member | Recruitment retention. |
| `interview_feedback` | HR | S/I after submit | tenant | interview + author | Restricted, immutable after submit except governed correction. |
| `onboarding_plan` | HR | C/versioned | tenant | tenant + key + version; draft → published/retired | Published immutable. |
| `onboarding_plan_step` | HR | I when published | parent-bound | plan + key | Retain with plan. |
| `onboarding_instance` | HR | S | tenant | employee + plan instance; pending → active → completed/cancelled | HR retention. |
| `onboarding_step` | HR | S | tenant | instance + plan step | HR/work retention. |

## 20. Cardinality and Optionality Rules

1. A User has zero or more Memberships; a Membership belongs to exactly one User and one Tenant.
2. A User can have at most one Membership per Tenant in v1.0.
3. A Membership can have multiple effective placements, but at most one primary placement at a given time.
4. Organization, Business Unit, Department, Team, Location, Cost Center, and Folder parent links are optional at root and acyclic.
5. A Role Assignment belongs to exactly one Membership and Role and has exactly one validated scope.
6. A Resource has exactly one Resource Type and Tenant; a securable aggregate root has exactly one Resource.
7. Resource Relationships connect two distinct same-tenant Resources valid for the declared Relationship Type.
8. A Record Value belongs to exactly one Record and one compatible Field Definition.
9. A published Workflow Version has exactly one initial state and one or more states; transitions connect states in that same version.
10. A Workflow Instance references exactly one Resource and one published Workflow Version.
11. An Approval belongs to one Workflow Instance and target Resource; a Decision belongs to one Approval Step and authorized actor.
12. A Document Version references one Document and one available File.
13. An Attachment references exactly one target Resource and exactly one source File or Document.
14. A Comment belongs to exactly one Resource; a Mention belongs to one Comment and same-tenant Membership.
15. Published Automation Versions have at least one Trigger and one Action.
16. One Domain Event has at most one Outbox Message in the same transaction.
17. A Plugin Installation is unique for Tenant and Plugin; it references exactly one active installed version.
18. CRM Lead Conversion has one Lead and zero or one target for each allowed resource type; a successfully converted Lead has at most one successful conversion record.
19. A Sales Discount Request references exactly one Quotation or Sales Order.
20. An Employee references zero or one Membership; one Membership references at most one Employee in the same tenant.
21. Effective-dated Employment and Compensation intervals for the same classification MUST NOT overlap unless explicitly allowed by policy.
22. An Application belongs to one Candidate and one Requisition; interview records remain bounded to that application.

## 21. Constraint Catalog

| ID | Constraint | Enforcement target |
|---|---|---|
| ERD-C001 | Every tenant-owned row has exactly one tenant. | NOT NULL tenant key or parent-bound composite FK. |
| ERD-C002 | Every tenant-owned FK is same-tenant. | Composite unique/FK pairs or validated trigger where polymorphic. |
| ERD-C003 | Tenant-local keys are unique only within their declared owner/scope. | Composite unique constraints. |
| ERD-C004 | Hierarchies cannot contain self-links or cycles. | Self checks plus closure/recursive validation. |
| ERD-C005 | Resource type + concrete aggregate ID is unique per tenant. | Unique constraint. |
| ERD-C006 | Generic references use `core.resource`; unconstrained type/ID pairs are prohibited. | Foreign keys and registration controls. |
| ERD-C007 | Role assignments have exactly one valid scope anchor compatible with scope type. | Discriminated checks plus scope-anchor integrity implementation. |
| ERD-C008 | Active effective intervals have valid bounds and prohibited overlaps are excluded. | Checks and exclusion/transactional constraints. |
| ERD-C009 | Reserved roles and registered permission keys cannot be tenant-mutated. | Privilege boundary and mutation guards. |
| ERD-C010 | Published workflow/object/automation/pricing/onboarding versions and children are immutable. | Status guards and update/delete prevention. |
| ERD-C011 | Exactly one current/active published version exists where the owner declares one. | Partial uniqueness/controlled transition. |
| ERD-C012 | A workflow transition's from/to states belong to the same workflow version. | Composite foreign keys. |
| ERD-C013 | Runtime workflow state belongs to the instance's workflow version. | Composite foreign key. |
| ERD-C014 | Audit, security event, domain event, revision, history, and decision facts are append-only. | Restricted privileges and mutation guards. |
| ERD-C015 | Outbox event identity/payload reference is immutable after insertion. | Mutation guard; status-only controlled updates. |
| ERD-C016 | Record values are unique per record/field and compatible with the published object version. | Unique plus typed/check/trigger contract. |
| ERD-C017 | Resource relationship endpoints are distinct, same-tenant, and compatible with relationship type. | Checks plus type validation. |
| ERD-C018 | Assignments identify one member or team according to assignment type. | Discriminated check. |
| ERD-C019 | Work dependencies cannot self-reference; configured dependency types are acyclic where required. | Check plus cycle validation. |
| ERD-C020 | Approval decisions are attributable to an authorized actor and cannot be rewritten. | FK, insert-only row, application authorization, audit. |
| ERD-C021 | Attachment has exactly one source: File XOR Document. | Check constraint. |
| ERD-C022 | Only validated/available files can back downloadable document versions. | State validation within transaction. |
| ERD-C023 | Mentioned/subscribed memberships and resource/comment share tenant. | Composite foreign keys. |
| ERD-C024 | Event IDs are unique; deliveries are unique per event/subscriber; idempotency keys are namespace-unique. | Unique constraints. |
| ERD-C025 | Plugin capability grants must be declared by the installed plugin version. | Composite relationship validation. |
| ERD-C026 | Raw secrets are absent from ordinary configuration and event/audit payloads. | Secret-reference model plus boundary validation. |
| ERD-C027 | CRM/Sales/HR links to other plugins use registered resources, not private-table FKs. | Schema dependency checks. |
| ERD-C028 | Quotation/order line numbers are unique within parent; monetary quantity/value constraints are valid. | Composite unique and numeric checks. |
| ERD-C029 | Discount Request references Quotation XOR Sales Order; Decision is append-only and SoD-authorized. | Check, FK, policy, audit. |
| ERD-C030 | Compensation and submitted interview feedback use restricted access and auditable reads. | RBAC mapping, classification, read audit. |
| ERD-C031 | Candidate retention/consent state controls archival/erasure; attribution facts retain non-sensitive references. | Retention workflow. |
| ERD-C032 | Optimistic-lock version increments atomically for mutable aggregates. | Update contract/trigger as decided at Gate 3. |
| ERD-C033 | Parent archive never cascades destructive deletion into material history. | Restrictive FK actions and archival workflows. |
| ERD-C034 | Derived authorization attributes never grant when stale or absent. | Transactional sync/version check and fail-closed query contract. |

## 22. Resource-to-Entity Traceability

| Approved RBAC resource | Primary entity/entities |
|---|---|
| `identity.user`, `session`, `credential`, `mfa_factor` | `user`, `session`, `credential`, `mfa_factor` |
| `organization.tenant` | `tenant` |
| `organization.organization`, `business_unit`, `department`, `team`, `position`, `location`, `cost_center` | Same-named organization entities |
| `organization.membership`, `team_membership` | `membership`, `membership_org_placement`, `team_membership` |
| `authorization.permission`, `role`, `role_permission`, `role_assignment` | Same-named authorization entities plus `service_role_assignment`, `role_assignment_event` |
| `configuration.tenant_setting`, `feature` | `tenant_setting`, `feature_setting` |
| `integration.connection`, `webhook` | `integration_connection`, `webhook`, `webhook_delivery` |
| `data.object_type`, `field_definition` | `object_type`, `object_type_version`, `field_definition`, `field_option` |
| `data.record`, `relationship` | `record`, `record_value`, `relationship_type`, `resource_relationship` |
| `search.query`, `saved_query` | Query service over `resource`; `saved_query` |
| `work.task`, `project`, `milestone`, `request`, `assignment` | Same-named work entities plus members/dependencies/history |
| `workflow.definition`, `version`, `instance`, `approval`, `delegation` | Workflow/approval entities in Section 10 |
| `content.document`, `document_version`, `file`, `folder`, `attachment` | Same-named content entities plus scan/closure/upload |
| `communication.comment`, `mention`, `notification`, `subscription` | Same-named communication entities plus revision/delivery |
| `automation.rule`, `execution` | Automation definition/version/runtime entities |
| `event.domain_event`, `outbox_message` | `domain_event`, `outbox_message`, subscription/delivery/idempotency entities |
| `audit.activity`, `audit_event`, `security_event` | Same-named observability entities |
| `plugin.catalog_entry`, `installation`, `capability_grant`, `configuration` | Plugin catalog/version/dependency/installation/capability/configuration/migration/lifecycle entities |
| `crm.account`, `contact`, `lead`, `opportunity`, `pipeline`, `activity` | CRM entities in Section 16; generic Activity plus `crm_activity_link` |
| `sales.quotation`, `quotation_line`, `sales_order`, `sales_order_line`, `pricing_rule`, `discount` | Sales entities in Section 17 |
| `hr.employee`, `employment`, `compensation`, `leave_request`, `requisition`, `candidate`, `application`, `onboarding` | HR entities in Section 18 |

Actions are application operations over these aggregates and do not each require a separate table. Immutable action outcomes are represented by the relevant history, decision, audit, event, execution, or lifecycle entity.

## 23. Invariant Traceability

| Invariants | ERD controls |
|---|---|
| INV-001–003 | Direct tenant ownership, same-tenant FKs, Resource registry, scoped authorization entities. |
| INV-004 | Plugin installation/capability model and no private cross-module foreign keys. |
| INV-005–008 | Secret references, append-only audit, Resource-based search filtering, tenant-aware execution context. |
| INV-009–012 | Opaque stable IDs, versioned object definitions, canonical/derived labels, module ownership rules. |
| INV-013–015 | Immutable published workflow graph, version-consistent runtime state, transition execution history. |
| INV-016–020 | Unique versioned Domain Event, tenant context, idempotency, Outbox and delivery state. |
| INV-021–023 | Work Assignment, explicit unassigned state, status history, attributable Approval Decision. |
| INV-024–027 | Plugin Version, Dependency, Migration, Capability Definition/Grant, Lifecycle Event. |
| INV-028–030 | File authorization anchor, Upload Session/File Scan, random object key. |
| INV-031–034 | Secret-reference configuration, release/security metadata boundary, retention-ready canonical data, server-owned constraints. |

## 24. Consequential v1.0 Modeling Decisions

1. A concrete `core.resource` registry is adopted as the generic integrity and authorization anchor.
2. Every tenant-owned relationship is physically expected to enforce same-tenant integrity in Gate 3.
3. Resource authorization projection fields are derived helpers, not canonical ownership truth.
4. User, Membership, and HR Employee remain separate entities.
5. Organizational placement is effective-dated; Position never grants a security Role.
6. Roles are flat; scoped role assignments are first-class effective-dated entities.
7. Dynamic records use versioned Object Types and Field Definitions; published definitions are immutable.
8. Workflow and automation runtime instances bind immutable published versions.
9. Approvals use explicit steps, assignees, delegations, and immutable decisions.
10. Cross-plugin links use `core.resource`; direct plugin-private dependencies are prohibited.
11. Audit/events/history remain when canonical resources are archived or legally erased, with sensitive payload minimization.
12. Compensation is a separate HR aggregate, and interview feedback has its own restricted entity.
13. Search, activities, notifications, hierarchy closure, and authorization projections are derived/rebuildable where identified.
14. Hard-delete cascades are not part of ordinary business lifecycle.

## 25. Deferred Physical Decisions for Gate 3

- UUID generation strategy and supported PostgreSQL version.
- Physical schemas/namespaces per module.
- Composite tenant foreign-key patterns and scope-anchor implementation.
- Typed custom field storage layout.
- RLS and transaction-local tenant context.
- Trigger versus privilege implementation for append-only/immutable entities.
- Closure-table maintenance mechanism.
- Exclusion constraints for effective-date overlaps.
- JSON schema validation boundary and safe payload limits.
- Indexes, partitions, generated columns, full-text search, and retention jobs.
- Database/application roles and plugin migration privileges.

No deferred item changes the logical entities, ownership, cardinalities, or security requirements in this ERD.

## 26. Gate 2 Review Summary

- Bounded contexts: 13
- Logical entities in authoritative Entity Register: 135
- Core/platform entities: 103
- CRM entities: 9
- Sales entities: 9
- HR entities: 14
- Generic cross-domain anchor: `core.resource`
- Direct cross-plugin private-table foreign keys: 0
- Constraint requirements: 34
- Domain invariants traced: 34

## 27. Approval Gate

| Gate | Artifact | Status | Approval date | Approver | Notes |
|---|---|---|---|---|---|
| Gate 1 | RBAC + Resource Permission Matrix v1.0 | Approved | 2026-08-29 | Product owner | Normative input for this ERD. |
| Gate 2 | Complete ERD v1.0 | Approved | 2026-08-29 | Product owner | Approved in conversation with `APPROVED — Complete ERD v1.0`. |
| Gate 3 | PostgreSQL Schema v1.0 | In progress | — | — | Authorized by approved Gate 2. |

Approval phrase: `APPROVED — Complete ERD v1.0`
