# ADR-019: Deployment Model

Status: Accepted
Date: 2026-08-29

## Decision
Support SaaS, private cloud, and on-premise deployment.

## Rationale
The target market may have different data residency, procurement, security, and infrastructure requirements.

## Rule
Deployment-specific configuration must not fork business logic.

## Consequence
Packaging, migrations, backup, upgrade, observability, and configuration must be designed for reproducible deployments.
