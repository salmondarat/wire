# CompanyOS Architecture Governance v1.0

## 1. ADR Process

Create an ADR when a decision:
- affects multiple modules
- changes security posture
- changes data ownership
- changes deployment model
- introduces a major dependency
- affects compatibility
- changes a core architectural invariant

## 2. ADR States

- Proposed
- Accepted
- Superseded
- Rejected
- Deprecated

## 3. ADR Template

```text
# ADR-NNN: Title

Status:
Date:

## Context

## Decision

## Alternatives

## Rationale

## Consequences

## Security Impact

## Migration / Rollout

## Related Decisions
```

## 4. Change Rule

Do not edit an accepted ADR to rewrite history.

If a decision changes:
1. create a new ADR
2. reference the previous ADR
3. mark the previous ADR superseded
4. document migration consequences

## 5. Architecture Review

A change must receive architecture review when it modifies:
- Core domain
- tenant model
- authorization
- plugin contract
- workflow engine
- event contract
- persistence strategy
- deployment model
- security boundary

## 6. Security Review

Mandatory for:
- authentication
- authorization
- file handling
- webhooks
- external HTTP calls
- secrets
- exports
- bulk operations
- plugin execution
- administrative operations

## 7. Versioning

Architecture documents:
MAJOR.MINOR

MAJOR:
breaking architectural change.

MINOR:
compatible addition.

Plugin APIs and event schemas require explicit compatibility policy.

## 8. Ownership

Each domain/module must have:
- technical owner
- documented boundary
- data ownership
- public contracts
- permission catalog
- event catalog
- test coverage

## 9. Review Checklist

- Does this violate a domain invariant?
- Does this weaken tenant isolation?
- Does this bypass authorization?
- Does this create hidden coupling?
- Does this create a new source of truth?
- Is the migration reversible?
- Is the operational impact known?
- Is observability sufficient?
- Are tests covering negative/security paths?
- Is there an ADR if required?

## 10. Engineering Rule

Prefer the simplest architecture that preserves the defined invariants and future evolution path.
