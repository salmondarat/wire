# ADR-001: Product Architecture

Status: Accepted
Date: 2026-08-29

## Context
CompanyOS must support many industries while keeping a stable common platform.

## Decision
Build CompanyOS as a platform with a stable Core and separately defined business plugins.

## Alternatives
1. One monolithic ERP containing every feature.
2. Separate application per industry.
3. Fully generic low-code platform.
4. Core + modular plugins.

## Rationale
Core + plugins balances reuse, configurability, maintainability, and commercial extensibility.

## Consequences
Positive:
- reusable platform
- clear product boundaries
- module-based monetization
- industry expansion without Core rewrites

Negative:
- plugin contracts must be carefully designed
- version compatibility becomes a product concern

## Security
Plugins cannot bypass Core security contracts.
