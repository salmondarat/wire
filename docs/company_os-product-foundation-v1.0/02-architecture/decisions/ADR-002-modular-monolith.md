# ADR-002: Initial Modular Monolith

Status: Accepted
Date: 2026-08-29

## Context
The product has many domains but is at an early stage. Microservices would increase operational complexity before boundaries are proven.

## Decision
Use a modular monolith with strict internal domain boundaries.

## Alternatives
- microservices
- serverless
- monolith without module boundaries

## Rationale
A modular monolith is cheaper to develop locally, easier to test, easier to deploy, and still allows later extraction.

## Consequences
Modules must not use arbitrary cross-module database access.
Service extraction remains possible because contracts are explicit.

## Security
A shared runtime does not remove authorization boundaries.
