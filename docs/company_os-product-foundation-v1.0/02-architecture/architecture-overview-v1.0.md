# CompanyOS Architecture Overview v1.0

## 1. Architecture Shape

CompanyOS starts as a modular monolith.

```text
Browser
  |
Reverse Proxy
  |
Web / API
  |
Application
  +-- Identity
  +-- Organization
  +-- Authorization
  +-- Record
  +-- Work
  +-- Workflow
  +-- Automation
  +-- Document
  +-- Notification
  +-- Activity
  +-- Audit
  +-- Event
  +-- Plugin
  |
Infrastructure
  +-- PostgreSQL
  +-- Redis/Valkey
  +-- MinIO
  +-- Mailpit in development
```

## 2. Architectural Layers

### Presentation
Web UI, admin UI, API clients.

### Application
Use cases, orchestration, authorization enforcement, transaction boundaries.

### Domain
Business rules, entities, value objects, domain services, state transitions.

### Infrastructure
PostgreSQL, object storage, cache, queues, mail, external integrations.

Plugins participate through explicit contracts rather than unrestricted internal access.

## 3. Module Dependency Rule

Allowed:
`UI -> Application -> Domain -> Infrastructure`

Business modules may depend on Core contracts.

Core must not depend on business modules.

Business module A must not directly mutate private persistence owned by business module B.

Cross-module interaction uses:
- public application contracts
- domain events
- integration events
- explicit shared primitives

## 4. Transaction Rule

A database transaction should contain one coherent consistency boundary.

Do not use a distributed transaction across modules.

Cross-module side effects should use reliable event publication/outbox patterns where required.

## 5. Event Rule

Domain facts are emitted after successful state changes.

External delivery is asynchronous.

Handlers must be idempotent.

Event schemas are versioned.

## 6. Read Model Rule

Search indexes, dashboards, analytics views, and caches are derived data.

They can be rebuilt from canonical sources where feasible.

## 7. Data Rule

PostgreSQL is the system of record for transactional data.

MinIO is the object store for files.

Redis/Valkey is used for ephemeral/cache/coordination workloads and must not become the sole source of durable business truth.

## 8. Scalability Rule

Scale the monolith vertically and horizontally first.

Extract a service only when a concrete requirement exists:
- independent scaling
- isolation
- security boundary
- independent deployment
- team ownership
- infrastructure constraints

## 9. Security Boundary

Every request must carry a resolved security context:
- authenticated principal
- tenant
- membership
- roles
- scopes
- permissions

Every resource access must be authorized at the server.

## 10. Deployment Boundary

Development:
application processes locally, infrastructure in containers.

Production:
all components may be containerized or packaged according to deployment target.

The runtime configuration contract remains environment-driven.

## 11. Architecture Quality Attributes

Priority:
1. security
2. correctness
3. maintainability
4. portability
5. observability
6. operability
7. performance
8. feature velocity

Performance optimization must not weaken tenant isolation or authorization.

## 12. Failure Philosophy

The system should fail closed for authorization.

For asynchronous workflows:
- retry transient failures
- use backoff
- use idempotency keys
- retain failure state
- expose operational status
- prevent duplicate business effects

For critical persistence:
- never silently discard writes
- surface errors
- preserve auditability
