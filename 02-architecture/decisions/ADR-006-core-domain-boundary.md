# ADR-006: Core Domain Boundary

Status: Accepted
Date: 2026-08-29

## Decision
Core owns reusable primitives:
identity, organization, authorization, records, relationships, work, workflow, documents, communication, events, automation, audit, search, configuration, plugin lifecycle.

Business meaning belongs to plugins.

## Rule
Core owns primitives. Plugins own business capabilities.

## Consequence
Core remains industry-neutral and avoids becoming a giant ERP module.
