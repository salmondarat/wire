# ADR-004: Identity Model

Status: Accepted
Date: 2026-08-29

## Decision
Separate User identity from Employee/business profiles.

Core concepts:
User, Credential, Session, MFA factor, Identity Provider, Membership, Profile.

## Rationale
This supports employees now and customers/vendors/partners later without redesigning authentication.

## Security
Passwords use Argon2id. Sessions are revocable. Authentication data is never exposed to plugins.
