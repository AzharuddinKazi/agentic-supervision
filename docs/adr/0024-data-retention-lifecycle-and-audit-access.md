# Data retention, lifecycle, and audit-access review

## Status

Accepted — 2026-09-16

## Context

An independent architecture review (`.scratch/architecture-review-2026-09-15.md`, Medium Finding #11) noted that ADR-0015 answers *where* data lives (on-prem) but nothing answers *for how long* or *who can look at it later*. That gap is sharpest for Langfuse: per ADR-0016, it captures the full prompt/completion transcript of every agent call — meaning sensitive supervisory content and LFI-identifying detail sits in a second store by default, with no retention period, alongside the Fraud Database's append-only audit trail (ADR-0007), which has a different and longer regulatory purpose.

## Decision

1. **Langfuse traces get a bounded retention period, shorter than the Fraud Database's audit trail.** Full prompt/completion content is only needed for a working evaluation window (debugging recent agent behavior, spot-checking human-agreement rate), not as a permanent record — the permanent record of what was decided is the audit trail, not the trace. The exact number of days is an open item to confirm with CBUAE's own data-governance function; until then, Langfuse traces are assumed non-permanent and the Fraud Database's `audit_log[]` is treated as the durable source of truth.
2. **The Fraud Database's audit trail persists per CBUAE's regulatory record-keeping requirements**, not a value this team picks unilaterally — flagged as an open item to confirm with compliance/legal, not assumed indefinite by default.
3. **A new RBAC role, Auditor/Compliance Reviewer, is added alongside Examiner and Lead/Approver (ADR-0007)** — read-only across the Fraud Database's `audit_log[]` and Langfuse traces, with no ability to trigger agents or resolve an `interrupt()`. Access under this role is subject to a periodic access review (cadence to be set by CBUAE's own access-governance process).
4. **A voided examination is never physically deleted.** Instead, a voided examination is flagged `voided` in its state and excluded from active Examiner Dashboard views/queues, but its `audit_log[]` entries remain queryable by the Auditor/Compliance Reviewer role. Actual erasure (e.g., a formal right-to-erasure request) requires a distinct, deliberate legal-hold-lift process outside the pipeline's normal operation — never an ad hoc delete triggered from the application.

## Consequences

The Auditor/Compliance Reviewer role is the role ADR-0016's audit-consumption story already assumed exists ("compliance/audit review happens as read-only queries...") but never actually named. A regulator's own AI tooling retaining full prompt/completion transcripts of supervisory findings with no defined retention period or access-review process is itself a governance gap in a compliance department's own system — the exact failure mode FPSD exists to catch when a regulated bank does it. Naming the role and a retention/voiding policy now, before implementation, is cheap; retrofitting an access model onto a store that's already been queried ad hoc for a year is not. (The `voided` state named here had no field, no transition, and no UI action until Round 20's Principal Engineer audit fix — `docs/decision-log.md` decision 125.)
