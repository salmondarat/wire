# ADR-010: Event Architecture

Status: Accepted
Date: 2026-08-29

## Decision
Use domain events and asynchronous processing for decoupled cross-module behavior.

## Event envelope
event_id, event_type, version, tenant_id, actor, resource, timestamp, correlation_id, causation_id, payload.

## Reliability
Use an outbox/reliable publication pattern when an event must be atomically associated with a database state change.

## Security
Events are tenant-scoped and must not expose secrets.
