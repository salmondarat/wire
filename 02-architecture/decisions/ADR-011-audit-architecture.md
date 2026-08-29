# ADR-011: Audit Architecture

Status: Accepted
Date: 2026-08-29

## Decision
Maintain append-only audit records for security-sensitive and material business changes.

## Minimum data
actor, tenant, action, resource, timestamp, request ID, correlation ID, result, before/after where safe, source context.

## Rule
Audit records are not editable by ordinary users and are not deleted through normal application flows.

## Privacy
Sensitive credentials, tokens, passwords, and secrets are never logged.
