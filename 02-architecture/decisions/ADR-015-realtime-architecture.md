# ADR-015: Realtime Architecture

Status: Accepted
Date: 2026-08-29

## Decision
Use WebSocket-based realtime delivery for notifications, work updates, activity streams, and selected workflow state changes.

## Rule
Realtime is a delivery mechanism, not the canonical source of truth.

## Reliability
Clients must reconcile with API state after reconnect.
