# EDM data complements gap analysis; it is not cross-checked against submitted-document claims

## Status

Accepted — 2026-09-15

## Context

The original workflow description frames EDM (quarterly fraud data) as something the team cross-references against submitted documents during gap analysis, which reads as "reconcile document claims against EDM figures and flag discrepancies." A future engineer reading the original brief's "cross-referencing" language would reasonably build a document-vs-EDM reconciliation check.

## Decision

We confirmed this isn't the right mechanism: EDM and LFI-submitted documents are non-overlapping datasets to a large extent, with only incidental overlap — not enough to justify building automated consistency cross-checking in v1. Instead, EDM data plays two narrower roles: it's additional context alongside document evidence during gap analysis, and it's an input to drafting supervision questions (including for the live examination meeting).

## Consequences

That reconciliation approach was considered and explicitly rejected as not worth the build given the actual data shape. This decision is extended, not reversed, by ADR-0011 (EDM's quantitative-analysis role) — a future reader must not conflate "EDM feeds quantitative analysis" with the cross-checking rejected here.
