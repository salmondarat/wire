# ADR-013: Search Architecture

Status: Accepted
Date: 2026-08-29

## Decision
Start with PostgreSQL-backed search and indexing. Introduce a dedicated FOSS search engine only when scale or search quality requires it.

## Security
Search must be permission-aware. Search results cannot reveal unauthorized records.

## Rationale
Avoid premature infrastructure complexity.
