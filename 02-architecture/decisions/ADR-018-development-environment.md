# ADR-018: Development Environment

Status: Accepted
Date: 2026-08-29

## Decision
Application source runs locally with hot reload. Stateful infrastructure runs through Docker Compose.

## Rationale
This keeps development fast while preserving production-like infrastructure dependencies.

## Baseline
pnpm + Turborepo; TypeScript; Next.js/React; NestJS; PostgreSQL; Redis/Valkey; MinIO; Mailpit.

## Rule
Docker rebuild is not required for ordinary source changes.
