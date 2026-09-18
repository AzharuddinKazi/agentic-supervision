# Two-tier checkpoint RBAC and an audit trail are built into v1, not deferred

## Status

Accepted — 2026-09-15. Scope note added after ADR-0024 (2026-09-16).

## Context

The human-in-the-loop checkpoints (ADR-0003) only mean something if a specific authorized person is doing the approving.

## Decision

v1 includes a lightweight two-tier role model for checkpoint resolution — examiner (drafts) versus lead/approver (signs off) — rather than a flat shared-login model. Alongside it, v1 also builds an append-only audit trail logging what the pipeline proposed versus what a human approved or overrode at each checkpoint.

Both were deliberately pulled into v1 rather than deferred like most other "nice to have" refinements (e.g. content-validation agent, unified review queue, notice versioning — see ADR-0002). The reasoning: this is a compliance department building a tool whose output feeds regulatory findings, so who-approved-what and why is closer to a baseline requirement than a UX nicety, and an audit trail is cheap to design in from the start versus expensive to retrofit onto an already-built approval flow.

## Consequences

**Scope note (added after ADR-0024, resolves Principal Engineer audit 4.4):** "two-tier" here, and every later "two-role RBAC" reference (including ADR-0029's), describes the checkpoint-resolution axis specifically — only Examiner and Lead/Approver ever raise or resolve an `interrupt()`. ADR-0024 later adds a third role, Auditor/Compliance Reviewer, but it is read-only and participates in no checkpoint, so it doesn't change this axis; it's a separate oversight axis layered on top, not a revision of the two-tier model this ADR describes. The system has three RBAC roles total; two of them participate in checkpoints.
