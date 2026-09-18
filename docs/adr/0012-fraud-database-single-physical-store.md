# Notice corpus, audit trail, RFI Store, and analysis output share one physical database

## Status

Accepted — 2026-09-15. Extended 2026-09-16 to add RFI Store as a fourth logical sub-store.

## Context

The architecture (and ADR-0005, ADR-0007) describe the notice corpus, the audit trail, and gap-analysis/findings-analysis output as distinct concerns. The current-state process documentation names a single **Fraud Database** that stores all AI-generated content and analysis, and separately replicates the source system's file/folder structure.

## Decision

We're adopting "Fraud Database" as one physical store housing all of these as logical sub-stores, rather than separate physical databases. The logical separation stays load-bearing — citation-grounding (ADR-0004) needs the notice corpus queryable per-clause, and the audit trail (ADR-0007) needs its append-only guarantee — but physically, one database is the right amount of infrastructure for v1's scale.

**RFI Store is the fourth logical sub-store, added explicitly here to close a gap an independent review caught** (`.scratch/architecture-review-2026-09-15.md`, Medium Finding #13). Decision 15 describes the RFI's *source format* as a structured Excel sheet — that's how it arrives and how it's read (`ID + clause` fields, no free-text extraction needed) — but its *storage engine*, once ingested, is the same Oracle Fraud Database as notice corpus and audit trail, not a standalone file left on disk.

## Consequences

This is worth recording because the architecture diagram draws these as separate boxes, which a future reader could mistake for separate physical services requiring their own provisioning/ops. ADR-0019's Fraud-Database-wide TDE-at-rest requirement already implicitly assumed the RFI Store extension; this ADR makes it explicit so the diagram and the prose agree: RFI Store is Fraud-DB-backed, Excel is an ingestion format, not a persistence model.
