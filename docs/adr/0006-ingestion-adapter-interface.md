# Shared-workspace ingestion is a pluggable adapter, not a direct SFTP integration

## Status

Accepted — 2026-09-15

## Context

The shared workspace is reachable over SFTP today (confirmed after initially assuming no API/SFTP access at all), and a "smart portal" with an API is planned to replace it later, though not yet concrete.

## Decision

Rather than write the pipeline against SFTP directly, we defined a small adapter interface — list documents, fetch document, get last-modified — with SFTP as the v1 concrete implementation.

## Consequences

This is a deliberate abstraction (not premature) because the replacement transport is a known, if unscheduled, future event: without the adapter boundary, the smart-portal migration would require touching every downstream consumer of document data instead of swapping one implementation. The SFTP folder structure 1:1 mirrors the shared workspace's fixed per-license-type structure, so the adapter needs no separate structural mapping layer for v1.
