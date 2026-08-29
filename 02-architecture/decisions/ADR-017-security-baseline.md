# ADR-017: Security Baseline

Status: Accepted
Date: 2026-08-29

## Decision
Use security-by-design with OWASP ASVS and OWASP Top 10 as primary application references, supported by NIST SSDF principles.

## Mandatory practices
- threat modeling
- secure coding
- server-side authorization
- input validation
- rate limiting
- secure session handling
- secret scanning
- SAST
- dependency scanning
- container scanning
- DAST
- security regression tests
- audit logging

## Release gate
Critical unresolved security findings block release.
