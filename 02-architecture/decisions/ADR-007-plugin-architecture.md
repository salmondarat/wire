# ADR-007: Plugin Architecture

Status: Accepted
Date: 2026-08-29

## Decision
Plugins are versioned packages with manifests, dependencies, permissions, migrations, APIs, UI contributions, events, workflows, automations, and lifecycle state.

## Lifecycle
available -> install -> validate -> enable -> configure -> active -> disable -> upgrade/uninstall.

## Security
Plugins receive explicit capabilities. They cannot access secrets or private state outside declared contracts.

## Future
Marketplace support is architecture-ready but not an MVP requirement.
