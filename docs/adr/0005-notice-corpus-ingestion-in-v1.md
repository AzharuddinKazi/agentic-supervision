# Notice corpus ingestion (including OCR) is built as part of v1, not assumed pre-existing

## Status

Accepted — 2026-09-15

## Context

The citation-grounding hard rule (ADR-0004) can't work without a searchable, per-clause-addressable notice corpus. We considered treating this as a dependency on some other team's existing repository, but no such queryable corpus exists today — notices are mostly digitized PDFs with some scanned images.

## Decision

We're building the ingestion step (PDF → text, OCR where needed → chunked/indexed per clause) ourselves, in v1. This is a one-time, then incrementally-updated job (re-run only when a notice is added or amended) rather than a recurring high-frequency pipeline, which keeps it bounded despite being a real component.

## Consequences

Notice versioning is explicitly out of scope for v1: the corpus holds only the current/latest version of each notice, so an examination always checks against what's in force today rather than what was in force during a historical period under examination. This is a known limitation, not an oversight — worth flagging if a past-period examination case is ever hit. (A resolution path — structured `effective_from`/`effective_to` columns, not a graph/RAG extension — is pre-specified in ADR-0027 for when this is picked up.)
