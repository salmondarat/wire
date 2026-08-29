# CompanyOS Product Blueprint v1.0

## 1. Product Identity

Name: CompanyOS
Category: Modular Company Operating System
Primary deployment models: SaaS, private cloud, on-premise
Development philosophy: FOSS-first, local-first, security-by-design
Initial architecture: Modular monolith
Primary goal: company-wide source of truth, work management, monitoring, workflow orchestration, and teamwork.

## 2. Vision

CompanyOS provides one operational context for a company while allowing each company to configure its own structure, workflows, records, and business modules.

The platform should answer:
- What is happening?
- Who owns it?
- What must happen next?
- What data supports the status?
- Who changed it?
- Which process depends on it?
- What requires management attention?

## 3. Product Principles

1. Industry-neutral core.
2. Core owns reusable primitives.
3. Plugins own business capabilities.
4. Canonical records are the source of truth.
5. Work is universal across modules.
6. Workflow is a core capability.
7. Server-side authorization is mandatory.
8. Tenant isolation is a security invariant.
9. Audit is append-only.
10. FOSS-first development.
11. Local development must be simple.
12. Production deployment must be designed from the beginning.
13. Security is part of the SDLC.
14. Avoid premature microservices.
15. Prefer explicit contracts over hidden coupling.

## 4. Problem

Companies commonly distribute operational truth across spreadsheets, email, chat, documents, departmental systems, and manual approvals. This creates inconsistent status, duplicated data, unclear ownership, delayed handoffs, and poor management visibility.

CompanyOS addresses the system-level problem: lack of shared operational context.

## 5. Product Boundary

### Core Platform

Core provides:
- identity
- organization
- membership
- roles and permissions
- records and relationships
- tasks
- projects
- requests
- workflow
- approvals
- activities
- audit
- events
- automation
- notifications
- documents
- search
- configuration
- integrations
- plugin lifecycle

### Business Modules

Examples:
- CRM
- Sales
- HR
- Inventory
- Procurement
- Service
- Asset Management
- Finance
- Manufacturing

### Industry Modules

Examples:
- construction
- education
- logistics
- professional services
- manufacturing
- hospitality
- agencies

## 6. User Types

### Platform administrator
Manages organization, users, security, modules, integrations, configuration.

### Management
Needs KPIs, approvals, cross-department visibility, alerts, reports, and company activity.

### Manager
Manages teams, workload, approvals, projects, tasks, and operational reports.

### Employee
Works from a unified work surface containing tasks, requests, approvals, documents, comments, and notifications.

### External user
Optional future role for customers, vendors, partners, and contractors. Architecture supports it; MVP does not expose it.

## 7. Core Concepts

### Tenant
Security and billing boundary for SaaS.

### Organization
Company structure inside a tenant.

### User
System identity. A user may be associated with an employee or external party.

### Membership
Connects a user to a tenant and organizational scope.

### Record
A canonical business data instance.

### Object type
Definition of a record type.

### Relationship
Typed link between records.

### Work
A unit of work such as task, request, project, milestone, or assignment.

### Workflow
State machine governing record or work lifecycle.

### Event
Immutable fact that something happened.

### Activity
Human-readable operational history.

### Audit event
Security/accountability record of a material change or sensitive action.

### Plugin
Installable extension with explicit dependencies, permissions, contracts, and lifecycle.

## 8. Universal Work Model

All modules may create work:
- Sales creates follow-up tasks.
- HR creates onboarding tasks.
- Finance creates approvals.
- IT creates service requests.
- Procurement creates purchase requests.

The employee sees these through `My Work`.

## 9. Source of Truth

Each critical business object must have a canonical record.

Dashboards, reports, search indexes, caches, and derived views are not canonical sources.

A canonical record has:
- tenant
- object type
- identifier
- owner where applicable
- state/status
- version
- timestamps
- relationships
- activity history
- audit history

## 10. Workflow

Workflow supports:
- states
- transitions
- conditions
- validation
- actions
- approvals
- delegation
- escalation
- rejection
- revision
- versioning

Example:

Draft -> Review -> Approved -> Processing -> Completed

A transition must be authorized and validated server-side.

## 11. Event and Automation

Events follow:
`event type + tenant + actor + resource + timestamp + correlation + versioned payload`

Automation follows:
`trigger -> optional condition -> action(s)`

Actions include:
- create record
- update record
- assign work
- notify
- request approval
- change state
- call webhook
- send email

Critical asynchronous work must be traceable and idempotency-aware.

## 12. Plugin Model

A plugin contains:
- manifest
- version
- dependencies
- permissions
- domain capability
- database migrations
- API contracts
- UI contributions
- events
- workflows
- automations
- configuration
- lifecycle hooks

Plugins cannot bypass authorization, access other tenants, mutate another module's private state, or alter audit history.

## 13. Security Baseline

Reference standards:
- OWASP Top 10
- OWASP ASVS
- OWASP API Security Top 10
- NIST SSDF
- CWE

Controls:
- Argon2id password hashing
- secure session management
- MFA-ready architecture
- server-side authorization
- tenant isolation
- strict validation
- rate limiting
- secure headers
- CSRF protection where applicable
- SSRF controls
- secure file upload
- secret scanning
- SAST
- dependency scanning
- container scanning
- DAST
- security audit trail
- backup and restore testing

## 14. FOSS Development Baseline

Preferred stack:
- TypeScript
- React / Next.js
- NestJS
- PostgreSQL
- Redis or Valkey
- MinIO
- pnpm
- Turborepo
- Docker / Compose
- Vitest
- Playwright
- Semgrep
- Gitleaks
- OSV-Scanner
- Trivy
- OWASP ZAP
- OpenTelemetry
- Prometheus
- Grafana
- Loki

Vendor-neutral contracts must be preserved.

## 15. Development Runtime

Infrastructure is containerized:
- PostgreSQL
- Redis/Valkey
- MinIO
- Mailpit

Application code runs with hot reload outside the infrastructure containers.

Expected developer flow:
1. start infrastructure once
2. run application dev servers
3. modify source
4. hot reload
5. run tests as needed

A source-code change must not require rebuilding every infrastructure image.

## 16. Git and Worktree

Protected:
- main
- develop

Branch types:
- feature/*
- fix/*
- security/*
- refactor/*
- docs/*
- chore/*
- release/*
- hotfix/*

No direct pushes to protected branches.

Feature work should use isolated worktrees where useful.

Commit types:
- feat
- fix
- security
- refactor
- test
- docs
- chore

## 17. CI/CD Baseline

Pull request checks:
- lockfile validation
- lint
- typecheck
- unit tests
- integration tests
- build
- dependency scan
- secret scan
- SAST
- container scan
- security tests

Deployment:
development -> CI -> build artifact -> staging -> verification -> production approval.

## 18. Deployment Models

### SaaS
CompanyOS operates hosting, backup, monitoring, updates, and support.

### Private cloud
Customer controls infrastructure while CompanyOS supplies application and support.

### On-premise
Customer runs CompanyOS in its own environment.

The application contract should remain consistent across deployment models.

## 19. Business Model

Target commercial model:
- platform subscription
- module subscription
- user/employee tier
- usage-based charges where appropriate
- implementation
- custom module development
- integration
- migration
- training
- enterprise support

Custom plugin ownership baseline:
CompanyOS owns platform/plugin framework. Customer-owned custom business implementation is handled contractually.

## 20. MVP

Core MVP:
1. tenant
2. organization
3. user
4. department
5. team
6. role
7. permission
8. custom object
9. custom field
10. record
11. relationship
12. task
13. project
14. request
15. workflow
16. approval
17. notification
18. comment
19. attachment
20. activity
21. audit
22. search
23. automation
24. event bus
25. plugin manager

Initial plugins:
- CRM
- Sales
- HR

## 21. MVP Acceptance

MVP must prove:
- modules can be installed without manual Core modification
- CRM and Sales can share canonical context
- HR can use Core workflow
- permission prevents unauthorized access
- events can cross module boundaries
- automation can execute actions
- audit captures critical changes
- tenant isolation is verified by tests
- full local development works
- CI validates pull requests
- production image can be built
- deployment configuration exists

## 22. Non-Goals for MVP

- full accounting ERP
- full manufacturing ERP
- public plugin marketplace
- external customer portal
- microservices decomposition
- every industry-specific workflow

## 23. Roadmap

Foundation -> Core -> Data -> Work -> Workflow -> Platform -> Plugin SDK -> CRM/Sales/HR -> Security hardening -> Deployment -> Marketplace/Industry extensions.

## 24. Definition of Done

A feature is complete only when applicable:
- requirements documented
- architecture reviewed
- threat model reviewed
- migration implemented
- API implemented
- authorization implemented
- validation implemented
- tests added
- security tests added
- audit/events added
- error handling implemented
- documentation updated
- CI passed
- scans passed
- review completed
