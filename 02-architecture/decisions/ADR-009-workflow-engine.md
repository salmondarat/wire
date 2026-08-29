# ADR-009: Workflow Engine

Status: Accepted
Date: 2026-08-29

## Decision
Implement workflow as a Core state-machine capability.

## Model
Definition -> Version -> State -> Transition -> Condition -> Action -> Approval.

## Requirements
- versioning
- permission checks
- validation
- approvals
- escalation
- delegation
- rejection/revision
- auditability

## Rationale
Sales, HR, procurement, IT, and custom modules need the same workflow primitive.
