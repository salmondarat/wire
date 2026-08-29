# ADR-012: File Storage

Status: Accepted
Date: 2026-08-29

## Decision
Use S3-compatible object storage, with MinIO as the FOSS-first development/self-hosting implementation.

## Security
Private buckets, random object keys, MIME/magic-byte checks, size limits, filename sanitization, authorization on download, malware scanning where available.

## Rationale
Object storage decouples large binaries from PostgreSQL while keeping deployment portable.
