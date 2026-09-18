# The Fraud Database stores document content, not just pointers — promoted out of a capacity-plan parenthetical

## Status

Accepted — 2026-09-17

## Context

ADR-0022's capacity placeholder noted, in passing, that whether the Fraud Database stores document content or just pointers into the Shared Workspace/Smart Portal was "not yet decided." An independent architecture review (Principal Engineer audit, `docs/reviews/2026-09-17-principal-engineer-audit.md`, Medium Finding 5.6) correctly flagged that this is not a capacity detail — it's a data-architecture decision with retention, encryption, and evidentiary consequences, and it deserves its own decision rather than a parenthetical.

## Decision

**The Fraud Database stores full document content for anything Intake Tracker marks `submitted`** — the actual bytes, not just a pointer/reference into the source system — once a document enters the tracked intake flow (ADR-0006/0012). Pointers into the Shared Workspace/Smart Portal are used only transiently, during the Ingestion Adapter's fetch, never as the durable reference.

**Why pointers-only is not an option here, given what's already decided elsewhere:**
- `CONTEXT.md` states the Shared Workspace **is being delisted**, replaced by an internally-built Smart Portal. A pointer-based audit trail whose evidence lives in a system already scheduled for delisting means the evidence underlying a signed-off regulatory finding can disappear from under the audit trail that's supposed to be its permanent record (ADR-0007, ADR-0024).
- ADR-0024 already requires audit-log entries and their underlying evidence to remain queryable for voided examinations and for retention/lifecycle purposes; that guarantee is meaningless if the evidence itself is a dangling pointer into a system that no longer holds it.
- ADR-0019 already requires the Fraud Database to encrypt data at rest — content storage sits under that same guarantee for free; a separate pointer scheme would need its own, currently unspecified, protection story for whatever system it points into.

## Consequences

**Consequence for ADR-0022's capacity numbers**: the "low hundreds of MB per examination, low-tens-of-GB/year" placeholder was explicitly conditioned on the pointers-only answer and is **no longer valid** now that content storage is the decision — ADR-0022 should be revisited with real per-document-size assumptions once the examiner team's actual document volume is known. This ADR settles the architecture question; it does not attempt to re-derive the capacity number, which depends on real usage data ADR-0022 already says doesn't exist yet.
