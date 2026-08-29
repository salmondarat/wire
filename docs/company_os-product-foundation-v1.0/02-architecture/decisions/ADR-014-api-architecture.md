# ADR-014: API Architecture

Status: Accepted
Date: 2026-08-29

## Decision
Use versioned REST APIs as the primary external API. Use internal application contracts for module communication.

## Requirements
authentication, authorization, validation, pagination, filtering, sorting, consistent errors, idempotency where needed, rate limiting, request IDs.

## Future
GraphQL or additional protocols may be added only when justified.
