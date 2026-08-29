# CompanyOS Core Domain Model v1.0

## 1. Domain Map

```text
CompanyOS
|
+-- Identity
|   +-- User
|   +-- Credential
|   +-- Session
|   +-- MFA
|   +-- Identity Provider
|
+-- Organization
|   +-- Tenant
|   +-- Organization
|   +-- Business Unit
|   +-- Department
|   +-- Team
|   +-- Position
|   +-- Location
|   +-- Cost Center
|
+-- Authorization
|   +-- Membership
|   +-- Role
|   +-- Permission
|   +-- Scope
|
+-- Core Data
|   +-- Object Type
|   +-- Field Definition
|   +-- Record
|   +-- Record Value
|   +-- Relationship
|
+-- Work
|   +-- Task
|   +-- Project
|   +-- Milestone
|   +-- Request
|   +-- Assignment
|
+-- Workflow
|   +-- Definition
|   +-- Version
|   +-- State
|   +-- Transition
|   +-- Condition
|   +-- Action
|   +-- Approval
|
+-- Content
|   +-- File
|   +-- Document
|   +-- Folder
|   +-- Version
|   +-- Attachment
|
+-- Communication
|   +-- Comment
|   +-- Mention
|   +-- Notification
|   +-- Subscription
|
+-- Automation
|   +-- Trigger
|   +-- Rule
|   +-- Action
|
+-- Observability
|   +-- Activity
|   +-- Audit Event
|   +-- Domain Event
|   +-- Security Event
|
+-- Plugin
    +-- Manifest
    +-- Version
    +-- Dependency
    +-- Capability
    +-- Configuration
    +-- Migration
    +-- Lifecycle
```

## 2. Identity Domain

### User
System identity.

Key properties:
- id
- status
- email/username according to identity policy
- created_at
- updated_at

### Credential
Authentication material.

Must never be exposed to business modules.

### Session
Server-recognized authenticated session.

Properties:
- session id
- user id
- created_at
- expires_at
- revoked_at
- device metadata where appropriate

### MFA
Future-ready factor model.

### Membership
Connects User to Tenant and organizational scope.

## 3. Organization Domain

### Tenant
Primary SaaS isolation and billing boundary.

### Organization
Company representation within a tenant.

### Business Unit
Optional high-level operating unit.

### Department
Functional organizational unit.

### Team
Operational group of users.

### Position
Job/role-in-company concept, distinct from security Role.

### Location
Physical or logical operating location.

### Cost Center
Financial/management classification.

## 4. Authorization Domain

### Role
Named set of permissions.

### Permission
Atomic action on a resource.

Naming convention:
`domain.resource.action`

Examples:
- crm.contact.read
- sales.order.create
- sales.order.approve

### Scope
Restriction applied to permission.

Examples:
- tenant
- organization
- business unit
- department
- team
- location
- owner
- record

## 5. Core Data Domain

### Object Type
Definition of a record category.

Examples:
- customer
- asset
- request
- custom object

### Field Definition
Describes a field:
- name
- type
- required
- default
- validation
- visibility
- indexing/search behavior

### Record
Canonical instance of an Object Type.

### Relationship
Typed connection between records.

Examples:
- customer owns opportunity
- project contains task
- employee belongs to team

Relationship metadata:
- source
- target
- relationship type
- cardinality
- optionality
- created_at

## 6. Work Domain

### Task
Atomic unit of work.

Core fields:
- title
- description
- assignee
- reporter
- team
- priority
- status
- start date
- due date
- source reference

### Project
Container for coordinated work.

### Milestone
Meaningful project checkpoint.

### Request
Formal work intake object that can be routed through workflow.

### Assignment
Represents ownership/responsibility.

## 7. Workflow Domain

### Workflow Definition
Named workflow for a resource type.

### Workflow Version
Immutable published version of a workflow definition.

### State
Allowed lifecycle state.

### Transition
Allowed state change.

### Condition
Predicate evaluated before transition/action.

### Action
Side effect executed by workflow.

### Approval
Decision task requiring one or more authorized approvers.

## 8. Content Domain

### Document
Logical business document.

### File
Binary object.

### Version
Immutable document revision.

### Attachment
Reference connecting a file/document to a record or work item.

### Folder
Logical organization of documents.

## 9. Communication Domain

### Comment
Human discussion attached to a supported resource.

### Mention
Reference to a user/team in communication.

### Notification
User-facing event requiring awareness.

### Subscription
Preference to receive changes from a resource/event category.

## 10. Automation Domain

Automation:
`Trigger -> Condition -> Action`

Triggers:
- event
- schedule
- state transition
- record creation/update

Actions:
- create/update record
- assign task
- notify
- request approval
- change state
- webhook
- email

## 11. Observability Domain

### Activity
Operational history intended for users.

### Audit Event
Security/accountability record.

### Domain Event
Immutable fact used for decoupled application behavior.

### Security Event
Authentication/authorization/security control event.

## 12. Plugin Domain

### Manifest
Identity and compatibility metadata.

### Dependency
Required Core/plugin capability.

### Capability
Explicit API/service access granted to a plugin.

### Migration
Versioned database change owned by plugin.

### Lifecycle
available -> installed -> validated -> enabled -> active -> disabled -> upgraded/uninstalled.

## 13. Aggregate Candidates

Initial aggregate boundaries:

### Identity aggregate
User + credentials/session references.

### Tenant aggregate
Tenant + organization configuration references.

### Membership aggregate
User membership and scoped role assignments.

### Record aggregate
Record + controlled values and relationship references.

### Work aggregate
Task/request/project state and assignments.

### Workflow aggregate
Workflow version + states + transitions.

### Approval aggregate
Approval decision process.

### Document aggregate
Document metadata + versions.

### Automation aggregate
Automation definition and rules.

Plugin installation has its own aggregate.

Exact aggregate boundaries must be validated during implementation.

## 14. Canonical Data Rules

Canonical data:
- tenant
- user
- organization structure
- roles/permissions
- records
- workflow state
- work ownership
- document metadata

Derived data:
- search index
- dashboard cache
- analytics aggregate
- notification projection
- activity projections where reconstructable

## 15. Cross-Domain Examples

### Sales order approval
Sales plugin creates Order -> Core Workflow evaluates approval -> Approval creates work -> approved transition emits event -> Finance/Inventory consume event.

### Employee onboarding
HR creates employee -> automation creates tasks -> documents attach -> workflow tracks completion -> activities/audit record material actions.

### IT request
Employee creates Request -> authorization scopes request -> workflow routes -> manager approval -> task assignment -> completion -> audit.

## 16. Domain Events

Examples:
- record.created
- record.updated
- record.archived
- task.assigned
- task.completed
- workflow.transitioned
- approval.requested
- approval.approved
- approval.rejected
- document.created
- document.version.created
- user.created
- membership.changed
- plugin.enabled

Business modules should use namespaced events:
- crm.contact.created
- sales.order.approved
- hr.employee.onboarded

## 17. Domain Error Principles

Errors should distinguish:
- validation failure
- authentication failure
- authorization denial
- missing resource
- conflict
- invalid state transition
- dependency failure
- rate limit
- internal failure

Do not leak sensitive internal details to clients.
