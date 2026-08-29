# ADR-016: Configuration

Status: Accepted
Date: 2026-08-29

## Decision
Separate:
- environment configuration
- platform configuration
- tenant configuration
- user preferences
- plugin configuration

Secrets are environment/secret-store concerns, not ordinary tenant configuration.

## Security
Secrets must not be stored in Git, logs, client bundles, or ordinary database fields without an appropriate secret-management design.
