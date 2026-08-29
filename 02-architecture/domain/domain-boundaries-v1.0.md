# CompanyOS Domain Boundaries v1.0

## Boundary Rules

1. Core never imports business-module domain logic.
2. Business modules may depend on Core contracts.
3. Business modules cannot directly modify another module's private tables.
4. Shared concepts belong in Core only when they are genuinely industry-neutral.
5. Events are preferred for decoupled cross-module reactions.
6. Synchronous calls are acceptable for explicit request/response contracts.
7. Every module declares owned resources.
8. Every module declares public API.
9. Every module declares emitted and consumed events.
10. Every module declares permissions.
11. Every module declares migrations.
12. A plugin cannot silently change Core behavior.

## Ownership Examples

Core owns:
- User
- Tenant
- Team
- Permission
- Task
- Workflow
- Approval
- Document
- Notification
- Audit

CRM owns:
- Lead
- Contact business extensions
- Opportunity
- Pipeline

Sales owns:
- Quotation
- Sales Order
- Sales pricing rules

HR owns:
- Employee business profile extensions
- Leave
- Recruitment
- Onboarding

Inventory owns:
- Item
- Warehouse
- Stock movement

## Cross-module example

Sales Order approval may emit:
`sales.order.approved`

Inventory consumes it to reserve/prepare stock.

Finance consumes it for financial processing.

Sales does not directly write Inventory's stock table or Finance's ledger.
