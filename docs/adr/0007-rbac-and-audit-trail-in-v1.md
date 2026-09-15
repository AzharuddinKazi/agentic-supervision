# Two-tier RBAC and an audit trail are built into v1, not deferred

Given the human-in-the-loop checkpoints (ADR-0003) only mean something if a specific authorized person is doing the approving, v1 includes a lightweight two-tier role model — examiner (drafts) versus lead/approver (signs off) — rather than a flat shared-login model. Alongside it, v1 also builds an append-only audit trail logging what the pipeline proposed versus what a human approved or overrode at each checkpoint.

Both were deliberately pulled into v1 rather than deferred like most other "nice to have" refinements (e.g. content-validation agent, unified review queue, notice versioning — see ADR-0002). The reasoning: this is a compliance department building a tool whose output feeds regulatory findings, so who-approved-what and why is closer to a baseline requirement than a UX nicety, and an audit trail is cheap to design in from the start versus expensive to retrofit onto an already-built approval flow.
