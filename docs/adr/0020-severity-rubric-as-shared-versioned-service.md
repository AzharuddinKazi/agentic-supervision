# Severity rubric is a shared, versioned service — not Findings Author's private tool

## Status

Accepted — 2026-09-15

## Context

An independent architecture review (`.scratch/architecture-review-2026-09-15.md`, High Finding #7) noted that `agent-specifications.md` listed `severity_rubric.lookup(finding)` under Findings Author's own tool list, even though `CONTEXT.md` and decision 16 already describe the rubric as UI-editable data, independent of any one agent. As specified, it read like an agent-owned dependency, which hides a real question: the rubric can be edited mid-examination-cycle (that's the entire point of making it UI-editable), and an examination's AG/pre-exit revision loop (ADR-0010) can run for weeks after a finding was first drafted. If the rubric changes between the original draft and a redraft, which version applies to the redraft? Nothing previously answered this.

## Decision

1. `severity_rubric.lookup(finding, rubric_version?)` is a shared platform service, not an agent-local tool — any agent or future service may call it, and it is versioned independently of any agent's release cycle.
2. Every `findings[]` record carries the `rubric_version` it was evaluated against, alongside `severity` and `deadline` (extends the shared-state table in `agent-specifications.md`).
3. **A redraft during the AG/pre-exit revision loop re-evaluates against the rubric version current at redraft time, not the original version** — and this is a rubric-version change to log, not a silent overwrite. We picked re-evaluation over pinning because the rubric is UI-editable specifically so severity/deadline policy can be corrected without a code change; pinning to a stale version would mean a corrected rubric never actually applies to findings already in flight, defeating the reason the rubric is editable at all.

## Consequences

The tradeoff — a finding's severity can change between draft and redraft purely because the rubric changed, with no underlying fact about the finding changing — is accepted, and must be visible: any redraft where `rubric_version` differs from the original draft's is called out explicitly to the Lead/Approver at the findings/severity sign-off checkpoint, not silently re-applied. Two findings of the same type in the same examination getting different severities/deadlines purely because the rubric was edited between them is a real regulatory-consistency risk if it happens invisibly; making the version explicit and the re-evaluation visible turns a silent-drift risk into a reviewable fact the human checkpoint (decision 1) actually sees.

This also fixes the enforcement gap ADR-0017 already established the pattern for: the guardrail in `agent-specifications.md` ("persisted `severity` must equal the last `severity_rubric.lookup()` return value") now additionally requires that the persisted `rubric_version` matches the version that lookup call actually used — a mismatch on either field rejects the write.
