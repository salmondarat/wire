# ADR-005: Authorization Model

Status: Accepted
Date: 2026-08-29

## Decision
Use RBAC combined with resource permissions and scopes.

Evaluation:
principal -> tenant membership -> roles -> permissions -> resource -> scope/context.

## Example
sales.order.approve may be scoped to department=Sales.

## Rationale
Role-only authorization is too coarse for company-wide operations.

## Security
Authorization is enforced server-side. Frontend visibility is not a security control.
