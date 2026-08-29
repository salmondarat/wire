# CompanyOS Decision Register v1.0

## Accepted baseline

| Area | Decision |
|---|---|
| Product | Industry-neutral modular platform |
| Core | Reusable company primitives |
| Business logic | Plugins/modules |
| Architecture | Modular monolith |
| Database | PostgreSQL |
| Cache/ephemeral | Redis or Valkey |
| Object storage | S3-compatible, MinIO-first |
| API | Versioned REST |
| Realtime | WebSocket |
| Workflow | Core state-machine engine |
| Events | Versioned domain events |
| Automation | Trigger/condition/action |
| Authorization | RBAC + resource permissions + scopes |
| SaaS isolation | Tenant-aware shared PostgreSQL baseline |
| Identity | User separated from Employee/profile |
| Audit | Append-only |
| Search | PostgreSQL-first, dedicated engine later if needed |
| Development | Local-first |
| Infrastructure | Docker Compose for local dependencies |
| Monorepo | pnpm + Turborepo |
| Security | OWASP ASVS/Top 10 + NIST SSDF-aligned |
| CI | Tests + SAST + secret/dependency/container/security scans |
| Deployment | SaaS + private cloud + on-premise |
| External users | Architecture-ready, not MVP |
| Marketplace | Architecture-ready, not MVP |
| Customization | Custom objects, fields, workflows, automations |
| Pricing | Platform + modules + tiers + implementation/support |
| Plugin ownership | Framework CompanyOS, custom implementation contractual |
