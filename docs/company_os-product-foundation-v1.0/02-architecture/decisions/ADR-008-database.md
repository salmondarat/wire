# ADR-008: Transactional Database

Status: Accepted
Date: 2026-08-29

## Decision
PostgreSQL is the primary transactional database.

## Rationale
It is FOSS, mature, relational, transactionally strong, extensible, and suitable for structured company data.

## Rule
Business truth belongs in PostgreSQL. Object binaries belong in object storage.
