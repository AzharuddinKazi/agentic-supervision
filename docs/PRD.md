# PRD: LFI Examination Pipeline

| | |
|---|---|
| **Status** | Pre-implementation (design complete, no code written) |
| **Author** | FPSD (CBUAE) design process, captured by agent-assisted design sessions |
| **Last updated** | 2026-09-19 |
| **Reviewers** | Independent architecture reviews: 2026-09-15, 2026-09-17 (x2), 2026-09-18 — see `docs/reviews/` |
| **Source of truth for decisions** | `docs/decision-log.md` (117 decisions, 18 rounds) and `docs/adr/0001`–`0032` |

This document exists because the design process (18 rounds of decisions, 32 ADRs, 3 independent architecture reviews) has outgrown any single conversation. If you are reorienting on "what are we actually building," start here. For *why* a specific call was made, follow the cross-references into `docs/decision-log.md`.

---

## 1. Objective

Give CBUAE's Fraud Prevention and Supervision Department (FPSD) a multi-agent software system that **augments** — not replaces — the examiner team through all three phases of a Licensed Financial Institution (LFI) compliance examination, cutting the manual effort in document intake tracking, clause-by-clause compliance review, and findings drafting, while keeping every regulatory judgment human-signed-off.

## 2. Background

FPSD examiners currently run LFI examinations almost entirely by hand:

1. **Pre-examination**: a 15–20 question RFI (Request for Information), each question tagged to a specific notice/clause, is sent to the LFI 30 days before the examination. The LFI uploads evidence documents to a shared workspace. Examiners manually track what's been submitted, then manually read every document against the cited notice clauses (plus cross-referencing quarterly fraud figures from an EDM) to find compliance gaps — tedious and error-prone at LFI-examination scale.
2. **Examination meeting**: examiners ask clarification questions live on-site, based on the gaps found. Minutes are taken; any further documents the LFI must submit are tracked afterward.
3. **Post-examination**: gaps are finalized into severity-rated findings with deadlines, written into a transmittal letter and pre-exit deck, and routed to the Assistant Governor (AG) for approval before being presented to the LFI as the exit outcome.

This is the process the pipeline automates the tedious parts of, without removing the examiner or the Lead/Approver from any judgment call. See `CONTEXT.md` for the full domain glossary (LFI, RFI, Notice, Clause, Supersession, Compliance Verdict, Gap Analysis, EDM, Finding, Fraud Database, etc.) — this PRD uses those terms as defined there.

## 3. Goals

- **G1 — Reduce manual document-tracking and clause-review effort** across all three examination phases, for a **bank-only** LFI population in v1 (decision 17).
- **G2 — Every compliance judgment is grounded or explicitly flagged for a human.** No compliance verdict is issued without a verbatim, exact-match citation to a real notice clause; where grounding isn't found with confidence, the system says so instead of guessing (ADR-0004 — the single most load-bearing rule in the design).
- **G3 — Preserve human authority at every consequential step.** Augmentation, not autonomy (ADR-0003): supervision questions, findings/severity, and anything going to the AG are all human sign-off gates, not agent-decided outcomes.
- **G4 — End-to-end coverage in v1**, not a partial slice: intake tracking, gap analysis, meeting support, findings drafting, and AG/pre-exit routing all ship together as a lean, bare-minimum pipeline (ADR-0002, reversing an earlier "intake-only" scope — decision 22).
- **G5 — Full auditability.** Every agent-proposed vs. human-approved/overridden decision at every checkpoint is captured in an append-only, tamper-evident audit trail from day one (ADR-0007).
- **G6 — Fit CBUAE's real deployment constraints**: on-prem, self-hosted LLM (no third-party model API), existing CBUAE SSO, Oracle for the data store (ADR-0013/0014/0015).

## 4. Non-goals (v1)

- **Not full autonomy.** The system never sends an RFI, never messages the LFI directly, and never finalizes a regulatory judgment without a human sign-off (ADR-0003, decisions 82, 100).
- **Not cross-checking EDM data against submitted documents.** EDM figures are analyzed on their own terms (quantitative analysis) and used as drafting context — never reconciled line-by-line against document claims (ADR-0009, ADR-0011).
- **Not license types beyond banks.** SVFs and other license types are deferred (decision 17).
- **Not document-plausibility judgment.** v1 "submitted" means present, non-empty, and correctly named/formatted — whether the *content* actually answers the question is a human judgment call in gap analysis, not a separate content-validation agent (decisions 18, 28).
- **Not LFI-facing communication.** No reminder emails, no automated RFI delivery, no automated further-submission requests to the LFI (decisions 19, 82, 100).
- **Not a unified cross-feature review queue.** Flagged items (grounding failures, supersession ambiguity) surface within their own feature context, not a consolidated inbox (decision 31).
- **Not notice version history.** Current-version-only clause lookups; effective-dated historical versions are a known, explicitly flagged v1 gap (decision 32, resolution path defined in decision 79).
- **Not in-app rendering of the transmittal letter / pre-exit deck**, and **not a unified in-app metrics/observability dashboard** — both are explicit, named v2 candidates, not dropped (decisions 102, 116).
- **Not adverse-media / news / consumer-complaint monitoring** — legally gated on CBUAE legal/compliance sign-off, fully on hold (ADR-0028).
- **Not multi-language.** v1 targets English-language notices, RFI documents, and meeting minutes only — an explicit decision pending FPSD confirmation before Notice Corpus ingestion begins in earnest, not a silent assumption (ADR-0031).

## 5. Users and roles

Three roles, distinguished by **what actions they can take**, not by separate UIs — everyone sees the same screens and data within an examination (decision 89, ADR-0029):

| Role | Can do |
|---|---|
| **Examiner** | Drafts and works the flow: reviews gap-analysis output, drafts supervision questions, records meeting minutes, drafts findings. |
| **Lead / Approver** | Everything an Examiner can do, plus signs off at every HITL checkpoint (supervision questions, further-submission requests, findings/severity, AG feedback, rubric edits, Notice Corpus edits, examination setup). |
| **Auditor / Compliance Reviewer** | Read-only. Sees the same full detail as the other two roles, but scoped portfolio-wide across all examinations rather than one at a time — this is their landing page (decision 106, ADR-0024). |

Identity federates to CBUAE's existing SSO (Azure AD / on-prem AD via OIDC/SAML) — roles are authorization claims on federated identity, not a new user store (decision 46; the specific IdP is an open item, decision 112).

## 6. User stories

- As an **Examiner**, I want to see at a glance which RFI questions still have missing or suspicious documents, so I don't have to manually cross-check a folder listing against the RFI spreadsheet.
- As an **Examiner**, I want every compliance gap the system flags to show me the exact clause it's citing, so I can trust or challenge the verdict without re-reading the whole notice.
- As an **Examiner**, I want to submit meeting minutes as plain text and have the system pull out what further documents are needed, so I don't have to re-read my own notes hunting for action items.
- As a **Lead/Approver**, I want to sign off on supervision questions and findings one at a time, so a single bad item doesn't force me to accept or reject an entire batch.
- As a **Lead/Approver**, I want to see the AG's raw feedback text side-by-side with the system's summary of it, so I can catch a misrepresentation before it triggers a redraft.
- As a **Lead/Approver**, I want to override an intake status by hand with a reason, for the rare document that's correct but doesn't match the expected filename convention.
- As an **Auditor**, I want a portfolio-wide, cross-examination audit view, so I can spot-check sign-off patterns without being granted access to work any individual examination.
- As anyone on the team, I want to open a portfolio home screen and immediately see which examinations are approaching an SLA breach, so nothing quietly stalls in a queue.

## 7. Requirements by phase

### Phase 1 — Pre-examination (Setup, Intake, Gap Analysis tabs)

- Lead/Approver creates an examination: picks the LFI, sets a start date and SFTP root path, and **uploads the RFI that was already sent to the LFI** (the app never generates or sends an RFI — decision 82). The upload auto-parses into an editable question table for confirmation (decision 85).
- The system tracks document submission status per RFI question against the fixed, per-license-type folder structure, via a pluggable Ingestion Adapter (SFTP today, Smart Portal API later, decision 14/87) — no per-examination transport picker.
- **Gap analysis** runs as a single batch step once intake is complete (decision 39), producing: compliance verdicts per document/clause pair, a supervision-question draft list, and areas of strength/weakness — using notice-corpus citations and EDM context, never EDM-vs-document cross-checking (ADR-0009).
- Every verdict without a confident, verbatim clause citation is flagged `insufficient_grounding`, routed to human review rather than guessed (ADR-0004).
- Two resolution actions for a flagged item (decision 95): accept as `insufficient_grounding` (becomes a supervision question), or manually assert a verdict with a human-supplied citation.
- Supervision questions are signed off **per-question**, not as a whole list (decision 97).

### Phase 2 — Examination meeting (Meeting tab)

- Examiner submits meeting minutes as a one-shot text input after the meeting (decision 98) — no live transcription.
- The system extracts `further_submission_requests[]` from the minutes and routes them through a **new, previously-missing HITL checkpoint** (decision 99) before the list is final.
- Further-submission documents flow through the same automated Intake Tracker as round-1 documents (decision 100) — examiners never manually track a second round.
- The LFI is never messaged directly by the app for further-submission requests (decision 100) — the examiner relays them.
- A human-judged **sufficiency check** gates the transition from further-submission receipt to findings drafting (decision 57) — this loop was a real gap the first independent review caught.

### Phase 3 — Post-examination (Findings, Approval tabs)

- Findings are drafted along two parallel, never-reconciled tracks: qualitative (notice/clause compliance narrative) and quantitative (EDM figures analyzed on their own terms) (ADR-0011).
- Findings are signed off **per-finding**, tracks shown in visually separate sections (decision 101).
- Severity/deadline comes from a shared, versioned rubric service; a redraft re-evaluates against the *current* rubric version, with any drift surfaced explicitly rather than silently applied (ADR-0020).
- Transmittal letter and pre-exit deck are template-filled from signed-off findings data and stay **download-only** in v1 — no in-browser rendering (decision 102, deferred to v2).
- AG status is a dropdown plus required free-text on `changes_requested`; the raw text and the system's AI-generated summary are shown **side-by-side**, with only the summary editable — this is the fix for a Critical finding where AG feedback could otherwise steer a redraft on the system's interpretation alone, unseen (decision 56, 104).
- The Phase-3 revision loop ("LFI has concerns → update letter/deck → re-check → proceed to exit") is **routine pipeline state**, not a rare edge case (ADR-0010) — it reuses the same Findings tab redraft/sign-off flow; there is no separate "Exit Deck" tab (decision 105).
- An examination auto-closes (Active → Completed) the instant it reaches `proceeded_to_exit` — no manual "close" action (decision 107).

### Cross-cutting

- **Portfolio home**: every active examination, with LFI, license type, current phase, SLA countdown/breach flag, open-checkpoint count, last-activity timestamp — filterable Active/Completed/All (decisions 81, 92).
- **Notice Corpus management is a full in-app capability** (reversed from an initial "offline/ops process" recommendation — decision 108): upload → auto-OCR + auto-segment into clauses → human review/correction before anything becomes citable (decision 113).
- **Rubric editor**: a simple current-state edit form, global per license type, Lead/Approver only; full version history stays queryable via the Audit Log, not a separate history browser (decisions 84, 110).
- **Audit Log**: identical full detail to every role within an examination; Auditor's distinction is portfolio-wide *scope*, not extra detail (decision 106).
- **In-app notifications only** for v1 (SLA breaches/backlog surface as portfolio-home counts) — no email infrastructure (decision 91). A degraded-sync banner appears only after sustained tool failure (3 consecutive missed intervals), not on every transient retry (decision 117).

## 8. HITL (human-in-the-loop) checkpoints

Every consequential agent output stops at a human checkpoint before it becomes binding. Full table with resolving roles lives in `docs/architecture/agent-specifications.md`; the checkpoints are:

1. Supervision-question sign-off (per-question) — Lead/Approver
2. Further-submission review (per-item) — Lead/Approver — **new in Round 18, no prior checkpoint existed for this artifact**
3. Sufficiency check (further-submission → findings) — human-judged gate
4. Findings/severity sign-off (per-finding) — Lead/Approver
5. AG Feedback Review (raw vs. AI summary, summary editable) — Lead/Approver
6. Rubric edits — Lead/Approver
7. Notice Corpus edits (post-OCR/segmentation) — Lead/Approver

Primary trust metric: **human-agreement rate** (how often a Lead/Approver accepts a draft unedited) — not raw accuracy. A rate near 100% is treated as an automation-bias warning sign, not a win (ADR-0016).

## 9. Success metrics

No production usage data exists yet, so these are directional, to be sharpened once real examinations run through the system:

- Reduction in examiner hours spent on manual intake tracking and first-pass clause review, per examination.
- Human-agreement rate per HITL checkpoint, tracked over time and by LFI (a sustained near-100% rate, or an anomaly correlated with a specific LFI's documents, are both explicit investigation triggers — ADR-0016, ADR-0030).
- Zero regulatory findings shipped on a fabricated or misquoted citation (this is a hard product requirement, not an aspirational metric — ADR-0004).
- SLA adherence at each HITL checkpoint (durations themselves are still open — see §11).

## 10. Risks called out explicitly (not silently accepted)

- **Prompt injection via LFI-authored content** (documents, meeting minutes, AG feedback are all adversary-influenceable text fed to LLM agents). Accepted as a bounded risk for v1: the citation-grounding rule limits the worst case to `insufficient_grounding` (never a fabricated citation), and every LLM output passes a HITL gate before it's binding. Explicitly does **not** cover subtle severity-steering or interpretation manipulation within an otherwise-plausible citation — flagged as a real gap, with a revisit trigger (pre-go-live security exercise, or an agreement-rate anomaly). See ADR-0030.
- **Language scope**: OCR quality, the exact-match citation guardrail's Unicode/script behavior, model selection, and document templates all assume English. This needs explicit FPSD confirmation before Notice Corpus ingestion begins in earnest — not an assumption to build past. See ADR-0031.
- **Evidence retention**: the Fraud Database stores full document content (not pointers) for anything marked `submitted`, specifically because the source Shared Workspace is being delisted — a pointer-only design would let regulatory evidence disappear out from under its own audit trail. See ADR-0032.

## 11. Open items (explicitly not yet decided — do not assume an answer)

These are tracked in full in `docs/decision-log.md` §"Open items"; the headline list:

- Independent model-risk/compliance sign-off process for the AI system itself, and a cost model — both organizational, flagged by the first review, still open.
- Specific on-prem platform (VMs vs. K8s/OpenShift) and specific LLM (Qwen Coder vs. gpt-oss-120b) — both named candidates, neither committed.
- CBUAE SSO/IdP product choice, and the `secrets_manager` product choice — both "ask CBUAE IT," not guessed.
- HITL checkpoint SLA durations (supervision questions, further-submission review) — to be tuned empirically once real cases exist.
- Supersession confidence-escalation threshold — same "tune empirically" treatment.
- Capacity/GPU sizing (ADR-0022) — explicit placeholder, needs real annual examination volume from the examiner team before procurement.
- Sanctions/watchlist screening tool contract — scoped as a direction (ADR-0028), not yet specified.
- Notice Corpus management screen's tool contract — scoped as a direction (decision 108), not yet specified.
- Whether any in-scope content is Arabic/bilingual (ADR-0031) — blocks nothing yet, but blocks Notice Corpus ingestion once that starts in earnest.

## 12. Explicit v2+ candidates (deferred, not dropped)

- In-app rendering of the transmittal letter and pre-exit deck (currently download-only).
- A unified in-app metrics/observability dashboard (currently Langfuse/BI-tool query-based).
- Content-plausibility validation (beyond presence/format checking).
- SVF and other non-bank license types.
- LFI-facing automated communication (reminders, RFI delivery).
- A consolidated cross-feature human-review queue.
- Notice effective-dating/version history.
- Adverse-media/news/consumer-complaint monitoring (legally gated, see ADR-0028).
- A narrow, flag-only draft-completeness pre-check before Findings Author sign-off (ADR-0026) — only if real usage shows a specific need.

## 13. Reference map

| Question | Where to look |
|---|---|
| "Has X already been decided?" | `docs/decision-log.md` — check before reopening anything |
| "Why was X decided this way?" | `docs/decision-log.md`, cross-referenced to the ADR |
| Terminology / glossary | `CONTEXT.md` |
| Per-agent contract (inputs/outputs/guardrails/definition-of-done) | `docs/architecture/agent-specifications.md` |
| Per-tool contract (schema/timeout/retry) | `docs/architecture/tool-contracts.md` |
| Diagrams (architecture, workflow, sequence) | `docs/architecture/`, browsable gallery at `docs/index.html` |
| Independent review findings | `docs/reviews/` |
| Technical design detail | `docs/TDD.md` |
