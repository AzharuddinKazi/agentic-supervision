# Examiner Dashboard userflow architecture

ADR-0008 committed to one unified app; this ADR settles the structure that app actually has, discovered by walking the Examiner/Lead-Approver/Auditor journey screen-by-screen (decisions 81–117).

**Two-level navigation.** A portfolio home screen lists all active examinations (LFI, phase, SLA/backlog counts) as the landing page; clicking into one opens a per-examination workspace behind a permanent sidebar of tabs (Setup → Intake → Gap Analysis → Meeting → Findings → Approval → Audit Log). Tabs unlock by data availability, not a locked phase-stepper, since earlier-phase artifacts stay live reference material later and the AG/pre-exit revision loops (ADR-0010) aren't strictly forward-only.

**RBAC is gated actions on one screen, not parallel UIs.** Examiner and Lead/Approver see identical data and layout in every tab; sign-off controls render only for Lead/Approver. Two role-specific UIs over the same state would drift out of sync for no real benefit. The Auditor role (ADR-0024) is the one exception in shape, not detail: it gets a portfolio-wide, cross-examination Audit Log view as its landing page instead of the per-examination workspace, since it holds no drafting/sign-off authority that would need one.

**Examination setup is upload, not generation.** The RFI is a document the LFI has already received outside this app; setup uploads it (auto-parsed into the question list, human-confirmed) rather than generating or sending one. This mirrors the app's general stance toward LFI-facing communication: the app tracks pipeline state and reviews drafted output, but never itself talks to the LFI — further-submission requests are relayed by the examiner manually, the same way the RFI itself was.

**Notice Corpus management is in-app for v1**, reversing the initial assumption that ingestion could stay an offline/ops process — there is no external owner for this data, so upload → auto-OCR/auto-segment → human-confirm has to be a real screen, following the same "auto-derive, human confirms" pattern used for RFI parsing (ADR-0004's grounding rule requires an explicit confirmed state before any clause is citable).

**New HITL checkpoint: Further-submission review.** Auditing the meeting flow surfaced a genuine gap — `further_submission_requests[]` (extracted from meeting minutes) had no sign-off checkpoint at all, distinct from the pre-meeting supervision-question sign-off and the later sufficiency check. Added to the HITL table in `agent-specifications.md`, same per-item accept/edit/reject pattern as every other checkpoint.

**Two-role RBAC (decision 35) was reconsidered and reaffirmed during this round**, not changed — worth recording here because the entire checkpoint/guardrail architecture above assumes it holds.

**What stays explicitly out of v1, with a reason**: an in-app metrics/observability dashboard (ADR-0016 already routes this through Langfuse/BI-tooling to avoid a second reporting surface — flagged as a future-scope candidate, not dropped); in-browser rendering of the transmittal letter/pre-exit deck (download-only for now, same reason — flagged as future scope, not dropped); email/push notifications (in-app surfacing only); and local user/role management (assumed CBUAE SSO, specific IdP an open item like `secrets_manager` and the deployment platform).

Full per-screen/per-tab detail lives in `agent-specifications.md`'s Examiner Dashboard section and its HITL checkpoint table, not duplicated here.
