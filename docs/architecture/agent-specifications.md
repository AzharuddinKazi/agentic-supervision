# Agent Specifications — LFI Examination Pipeline v1

Detailed contract for every agentic and non-agentic component in `docs/architecture/lfi-pipeline-v1.html` and `lfi-pipeline-deployment-v1.html`. Each agent is a LangGraph node (or small subgraph); this document is the source of truth for what to actually build — inputs, outputs, tools, model behavior, state, human checkpoints (with SLAs), and the observability/evaluation contract each agent must satisfy. Read `CONTEXT.md` for vocabulary and `docs/adr/` for the decisions behind these contracts before changing them.

**The actual LangGraph node/edge/conditional-transition structure — not just this prose — lives in three workflow diagrams**, one per phase: `lfi-workflow-phase1-preexam-v1.html`, `lfi-workflow-phase2-meeting-v1.html`, `lfi-workflow-phase3-postexam-v1.html`. Read the relevant one before implementing a phase; it shows every guardrail branch, human checkpoint, and (Phase 2) the sufficiency-check gate that this document's prose alone does not make unambiguous.

**Every tool named below has a real input/output/timeout/retry contract in `docs/architecture/tool-contracts.md`** — this document names which tool each agent calls and why; that one specifies exactly what calling it looks like, including its distinct error/timeout shape (never collapsed into "no result found," ADR-0018).

## Shared graph state

All agents read and write a single LangGraph state object, scoped per examination (one instance per LFI per examination cycle). Fields referenced below.

**Examination record** (Principal Engineer audit 6.1 — these are the identity/lifecycle/scheduling fields the Examiner Dashboard already reads in `agent-specifications.md`'s own Dashboard section and in ADR-0029, but that were missing from this table; added here rather than left for an engineer to invent):

| Field | Written by | Read by |
|---|---|---|
| `lfi_id`, `license_type` | Setup (Lead/Approver, at creation) | Portfolio Home, `severity_rubric.lookup()` scoping (ADR-0020) |
| `examination_status` (`active` / `completed`) | Setup sets `active`; Approval Coordinator sets `completed` on `preexit_status = proceeded_to_exit` | Portfolio Home's Active/Completed/All filter |
| `voided` (boolean, default `false`) | Lead/Approver, explicit void action (ADR-0024 §4) | Portfolio Home (excludes voided from active views/queues); Audit Log (voided examinations' `audit_log[]` stays queryable regardless) |
| `current_phase` (`pre_examination` / `examination` / `pre_exit`) | orchestrator, on phase transition | Portfolio Home |
| `assigned_examiner`, `assigned_lead_approver` | Setup, at creation (editable per decision 111) | Examiner Dashboard RBAC scoping — "reach it only within an examination they're assigned to" (decision 106) |
| `model_version` (string) | orchestrator, pinned at examination creation | every LLM agent (ADR-0023 §4 — in-flight examinations stay on the model version they started under, never silently switched mid-cycle) |
| `start_date` | Setup | Portfolio Home, SLA calculations |
| `sftp_root_override` (optional) | Setup, only if this examination's path deviates from the LFI registry's default | Ingestion Adapter — **resolves audit finding 3.4**: the LFI registry (`agent-specifications.md`'s Settings/Admin section) holds the canonical per-LFI SFTP root; this field is an explicit per-examination override, checked first, falling back to the registry value. Setup's sequence diagram writes here only when an override is actually set, not on every creation |
| `rfi_questions[]` (ID, question text, notice ref, clause ref, `source: "rfi"\|"further_submission"`, `round: int`) | RFI Parser writes the `round: 1, source: "rfi"` entries at Setup confirm; Further-Submission Review appends `round: 2+, source: "further_submission"` entries when its checkpoint finalizes (**resolves audit finding 2.7** — a further-submission request becomes a trackable expected-document row the same way an original RFI question is, closing the "round-2 documents aren't RFI questions" gap) | Intake Tracker (expected-document list, both rounds), Compliance Analyst (round 1 only — round 2+ never re-triggers Gap Analysis, see Intake Tracker's trigger note below). Distinct from `rfi_responses[]` below, which is what the LFI actually submitted against this list |
| `open_checkpoints[]` (`{ checkpoint_type, opened_at, due_at }`) | whichever agent raises the `interrupt()` for that checkpoint; cleared on resolution | Portfolio Home's open-checkpoint count and SLA breach flag |
| `last_activity_at` | every agent, alongside its `audit_log[]` write | Portfolio Home |
| `meeting_scheduled_at` | **no current writer — open item.** The Meeting tab's sufficiency-check SLA is a hard deadline (the meeting date), not a duration (`agent-specifications.md`'s HITL table, "Suggested SLA" column) and nothing today captures when the meeting is scheduled. Flagging rather than inventing a scheduling screen not otherwise scoped; needs its own decision before Phase 2 build starts on this specific field, not blocking the rest of Phase 2 |

**Phase-flow state** (unchanged from the original table):

| Field | Written by | Read by |
|---|---|---|
| `schema_version` (integer, ADR-0025) | set at state creation; bumped only by a tested migration | every agent, before reading any other field |
| `rfi_responses[]` | Intake Tracker | Compliance Analyst |
| `intake_status` (per RFI question: submitted / pending / suspicious) | Intake Tracker | Examiner Dashboard |
| `compliance_verdicts[]` | Compliance Analyst | Examiner Dashboard, Meeting Facilitator |
| `supervision_questions[]` | Compliance Analyst, Meeting Facilitator | Examiner Dashboard, Audit Trail |
| `meeting_minutes` (raw text) | Examiner Dashboard (human input) | Meeting Facilitator |
| `further_submission_requests[]` | Meeting Facilitator | Intake Tracker (next cycle) |
| `meeting_followup_status` (sufficient / insufficient) | Meeting Facilitator, via Lead/Approver sufficiency check | Findings Author (its actual trigger condition), Examiner Dashboard |
| `findings[]` (severity, deadline, `rubric_version`, citations) | Findings Author | Examiner Dashboard, Approval Coordinator |
| `transmittal_letter_draft`, `pre_exit_deck_draft` | Findings Author | Approval Coordinator |
| `ag_status` (drafted / awaiting_ag / changes_requested / ag_feedback_reviewed / approved) | Approval Coordinator | Examiner Dashboard |
| `preexit_status` (scheduled / concerns_raised / resolved / proceeded_to_exit) | Approval Coordinator | Examiner Dashboard |
| `audit_log[]` (append-only) | every agent, on every state transition | Audit Trail Store |

State is persisted to the Fraud Database (ADR-0012) after every node execution — this is what makes LangGraph's interrupt/resume (ADR-0001) durable across the 30-day RFI window and the days/weeks between examination phases. Any breaking change to this table's shape bumps `schema_version` and ships with a migration function tested against an in-flight checkpoint (ADR-0025) — a deploy must not assume no examination is mid-flight under an older schema. Adding the Examination Record block above is itself such a change and must ship with a migration that backfills these fields for any examination already in flight when this schema version rolls out.

### `findings[]`, `compliance_verdicts[]`, `supervision_questions[]` element schemas

(Principal Engineer audit 6.3 — named as arrays above but never given per-element field lists, while `tool-contracts.md` already references named types for them that were defined nowhere.)

- **`FindingSummary`** (input to `severity_rubric.lookup()`): `{ finding_type: string, quantitative_flag: boolean, edm_ref?: EdmQueryRef }` — the minimal shape the rubric needs to classify a finding, before severity/deadline are assigned.
- **`FindingRecord`** (a `findings[]` element, what `template_fill` requires): `{ id: string, finding_type: string, description: string, severity: "low"|"medium"|"high", deadline_days: int, rubric_version: string, citations: Citation[], quant_ref?: EdmQueryRef, status: "draft"|"signed_off"|"redrafted" }`.
- **`Citation`** (the structure ADR-0004's grounding guardrail checks against): `{ clause_id: string, notice_ref: string, verbatim_text: string, sentence_index: int }` — `verbatim_text` is what the exact-match guardrail compares against the corpus; storing `sentence_index` alongside it lets a redraft re-verify against the same sentence rather than re-matching the whole clause.
- **`EdmQueryRef`**: `{ lfi_id: string, period_range: [string, string], metric: string, value: number }` — the traceable pointer `quant_analysis.compute()` returns and every quantitative finding must cite (ADR-0021's guardrail).
- **`ComplianceVerdict`** (a `compliance_verdicts[]` element): `{ question_id: string, verdict: "compliant"|"non_compliant"|"insufficient_grounding", citation?: Citation, asserted_by: "model"|"human" }` — `asserted_by` is what lets a human-asserted verdict (decision 95) log distinctly from a model-asserted one, per `lfi-gap-analysis-resolution-v1`'s requirement.
- **`SupervisionQuestion`** (a `supervision_questions[]` element): `{ id: string, question_text: string, source_verdict_id: string, disposition: "pending"|"accepted"|"edited"|"rejected" }` — see the reject-path note in the Further-Submission Review / Findings sign-off sections below for what each disposition value does downstream.

---

## Phase 1 — Pre-Examination

### Intake Tracker
**Persona**: none — **not an LLM agent**, a deterministic rule-based service. Included here because it gates Gap Analysis.

- **Trigger**: scheduled poll of the Ingestion Adapter (e.g., every 15 min) during the 30-day RFI window, and on-demand via Examiner Dashboard refresh.
- **Inputs**: `rfi_store` (expected question → folder/filename convention, per decision 15), Ingestion Adapter's `list_documents()` output.
- **Outputs**: `intake_status` per RFI question — `submitted` (present, non-empty, filename/format matches) or `pending` (absent). `suspicious` (present but fails the format check — decision 18) is **transient, never terminal**: the Intake tab's Lead/Approver review resolves every `suspicious` item one of two ways — **override** to `submitted` (a false negative: scanned doc, renamed file, decision 93) or **request resubmission**, which reverts it to `pending` and logs a distinct `resubmission_requested` audit event (a confirmed true positive — the file is actually wrong; fixes Principal Engineer audit 2.11, which found no path out of `suspicious` other than asserting it's fine). A `suspicious` item left unresolved simply stays `suspicious` and blocks `all_submitted` exactly like `pending` does — it is never silently treated as good enough to proceed.
- **Tools**: `ingestion_adapter.list_documents(lfi_id)`, `ingestion_adapter.get_metadata(doc_id)` (size, filename, last-modified).
- **Trigger for Gap Analysis**: fires the `all_submitted` event only when every **round-1** RFI question's status is `submitted` — `suspicious` and `pending` both block it (decision 39 — batch, not incremental). **Resolves audit finding 2.6**: the "Definition of done" section below previously stated a looser "100% non-`pending`" condition that would let unresolved `suspicious` items through; that was the wrong reading and is corrected there to match this one, the actual trigger Gap Analysis is built against.
- **Trigger for Meeting Facilitator's sufficiency check**: fires a separate `round_submitted(round: N)` event when every `rfi_questions[]` entry tagged `round: N` (N > 1, i.e. further-submission rows) is `submitted`. This is deliberately **not** the same event as `all_submitted` — round 2+ intake must not re-trigger Compliance Analyst's full Gap Analysis over the whole examination, only feed the Sufficiency Check gate that already exists for exactly this purpose.

### Compliance Analyst
**Persona**: **Compliance Analyst** — meticulous, cites chapter and verse, and says "I can't confirm this" rather than guess. System-prompt framing should make refusal-to-guess feel like professional competence, not failure, since the grounding rule (below) depends on the model being comfortable abstaining.

- **Trigger**: `all_submitted` event from Intake Tracker.
- **Inputs**: `rfi_responses[]` (documents), Notice Corpus (clause citations), EDM (fraud-data context, decision 21).
- **Outputs**: `compliance_verdicts[]` and the three Phase-1 artifacts — **supervision questions**, **LFI gaps and findings**, **areas of strengths and weaknesses** (decision 67 in `CONTEXT.md`'s Gap Analysis entry).
- **Tools**:
  - `notice_corpus.search(query, clause_filter)` — semantic + exact-match retrieval over the per-clause index.
  - `notice_corpus.get_clause(clause_id)` — exact verbatim sentence lookup, required before any verdict (ADR-0004).
  - `edm.query(lfi_id, period)` — read-only, context only, never written back.
  - `supersession_graph.check(clause_id)` — returns superseding clause + confidence score (decision 5, 30).
  - **Every tool call above retries transient failures (timeout, connection error) with backoff (ADR-0018) — a tool-unavailable result is never treated as "no matching clause/data," which would silently misfire the grounding rule below.**
- **Model behavior contract (hard rules, not prompted preferences — enforce in code, not just the prompt)**:
  1. **No verdict without a grounded citation.** If `notice_corpus.get_clause()` returns no high-confidence exact match (a real absence, not a timeout — see ADR-0018), the verdict is `insufficient_grounding` — never a best-guess compliant/non-compliant call (ADR-0004).
  2. Supersession confidence below the configured threshold (decision 30 — tune empirically; start conservative) routes that clause pair to `needs_human_review` rather than resolving it silently.
  3. In the workflow diagram (`lfi-workflow-phase1-preexam-v1.html`), both escalation reasons above are drawn as one merged "Human Review Flags" state — they route to the same Examiner Dashboard queue but must still be logged as distinct reasons in `audit_log[]`.
- **LangGraph shape**: a ReAct-style subgraph (retrieve → verify citation → verdict) looped once per RFI question/clause pair, not a single call over the whole document set — keeps each verdict independently inspectable and re-runnable.
  - **Sequencing note** (resolves Principal Engineer audit 3.6, and the apparent disagreement with the 2026-09-15 review's per-agent tool analysis — the two were never actually in conflict, just imprecisely worded): `notice_corpus.get_clause()` runs first and must return a high-confidence exact match before anything else happens — this is the raw text match, not yet a finalized verdict. `supersession_graph.check()` runs next, against that matched clause. **The verdict is finalized, and `log_checkpoint()` called, only after both steps complete** — so supersession is checked before the citation is finalized (the 2026-09-15 review's framing), even though it runs after the raw clause match is confirmed (`lfi-guardrail-citation-grounding-v1`'s framing). Running it before `get_clause()` would waste a lookup on a clause that might not even resolve; running it after the verdict is finalized would let a superseding clause invalidate a citation already logged as grounded. Both statements describe the same order — they were resolved by making them precise, not by changing the order.
- **Human checkpoint**: none at this node itself; `insufficient_grounding` and `needs_human_review` items surface in the Examiner Dashboard's gap-analysis review screen for the examiner to resolve before supervision questions are finalized.

---

## Phase 2 — Examination (Onsite)

### Meeting Facilitator
**Persona**: **Meeting Facilitator** — drafts sharp, specific questions an examiner could ask cold, and later reads meeting minutes the way a careful note-taker would: extracting commitments, not paraphrasing small talk.

Two distinct responsibilities, both owned by this node (decision 25):

**(a) Pre-meeting drafting**
- **Trigger**: gap analysis complete and examiner-reviewed.
- **Inputs**: `compliance_verdicts[]`, `supervision_questions[]` (draft), EDM context.
- **Outputs**: finalized `supervision_questions[]` list, formatted for the live meeting.
- **Human checkpoint**: **required sign-off** before the list is usable in the meeting (decision 1). LangGraph `interrupt()` here; resumes on Lead/Approver approval via Examiner Dashboard.

**(b) Post-meeting extraction**
- **Trigger**: `meeting_minutes` text submitted via Examiner Dashboard (human-typed, decision 25 — no audio/video transcription in v1).
- **Inputs**: `meeting_minutes`, `supervision_questions[]` (to match answers back to questions).
- **Outputs**: `further_submission_requests[]` — action items that route back to the Intake Tracker for the next document-collection cycle (same Shared Workspace/Smart Portal path, per the "v1 scope" card in the architecture diagram — deliberately not drawn as a separate integration).
- **Tools**: no external tool calls — pure extraction over the provided text, single LLM call with structured output.

**(c) Sufficiency check — added after architecture review (Critical Finding #3)**
`current_state.png`'s Phase 2 shows an explicit, iterative "documents/answers sufficient?" loop that earlier drafts of this spec left unmodeled, treating post-meeting extraction as one-shot with an undefined "further-submission loop resolved" condition. This is now a real, named gate (`lfi-workflow-phase2-meeting-v1.html`'s "Sufficiency Check" node):
- **Trigger**: `further_submission_requests[]` outcomes received back through Intake Tracker's next cycle.
- **Decision maker**: **Lead/Approver, not the LLM** (ADR-0003's augmentation principle — this is a judgment call about whether the LFI's response actually closes the gap, not an extraction task).
- **Outputs**: `meeting_followup_status` (`sufficient` / `insufficient`). `insufficient` re-enters Intake Tracker's document-collection cycle for round N+1, tracked via new `rfi_questions[]` rows (`round: N+1, source: "further_submission"`, see Shared graph state above) and gated by Intake Tracker's `round_submitted(round: N+1)` event, **not** `all_submitted` — that event is Phase 1's Gap Analysis trigger specifically and must not re-fire on further-submission rounds (resolves audit finding 2.7). `sufficient` is the condition that actually satisfies Findings Author's trigger below — "further-submission loop resolved" now means exactly this state value, not a prose assumption.

### Findings Author
**Persona**: **Findings Author** — writes for a reader (the Assistant Governor, then LFI leadership) who wasn't in the room: every finding stands on its own, with severity and citation, no institutional memory assumed.

- **Trigger**: `meeting_followup_status == sufficient` (the Meeting Facilitator's sufficiency check, above — not an assumed "resolved").
- **Inputs**: `compliance_verdicts[]`, `further_submission_requests[]` outcomes, EDM (for quantitative track).
- **Outputs**: `findings[]` split into two parallel analysis tracks (ADR-0011):
  - **Qualitative**: notice/clause compliance narrative — carries forward each finding's grounding citation.
  - **Quantitative**: EDM figures analyzed on their own terms (loss ratios, volume trends) — **never** reconciled against document claims (ADR-0009/0011 — this is the one rule most likely to be broken by a well-intentioned future engineer; do not add a "compare to document" tool here).
  - `transmittal_letter_draft` and `pre_exit_deck_draft`, filled into the fixed Word/PPT templates (decision 9) from `findings[]` — the deck is a derived artifact off the same data, not independently authored (decision 26).
- **Tools**: `severity_rubric.lookup(finding, rubric_version?)` → severity + deadline + the rubric version actually used (ADR-0020 — this is a **shared, versioned platform service**, not an agent-owned dependency: decision 16's UI-editable rubric can change mid-cycle, so every finding records which version it was evaluated against); `quant_analysis.compute(lfi_id, period_range, metric)` → deterministic ratio/trend/delta (ADR-0021 — **code, not the LLM**, does this arithmetic; the LLM only narrates the returned number); `template_fill(template_id, findings[])`. Full contracts (schema/timeout/retry) in `docs/architecture/tool-contracts.md`.
- **Guardrails**:
  - **(ADR-0017/ADR-0020)** Persisted `severity` and `rubric_version` must equal the last `severity_rubric.lookup()` call's return values for that finding — checked in code before the write, not left to the agent's narration. A mismatch on either field rejects the write. A redraft (AG/pre-exit revision loop) re-evaluates against the rubric version current at redraft time, not the original draft's version (ADR-0020); if that version differs from the original, the sign-off checkpoint below must surface this explicitly to the Lead/Approver, not apply it silently.
  - **(ADR-0021)** No quantitative finding may be persisted with a number that is not traceable to a specific `quant_analysis.compute()` return value — same enforcement pattern as the severity guardrail.
- **Human checkpoint**: **required sign-off** on findings/severity before the transmittal letter is drafted (decision 1, decision 26). `interrupt()` here.

### Approval Coordinator
**Persona**: **Approval Coordinator** — a state-tracker, not a persuader; it never drafts arguments to win over the AG or the LFI, it only tracks what was asked for and routes redrafts. Where an LLM is involved (summarizing feedback into redraft notes), it stays strictly descriptive.

Two loops, both routine state (ADR-0010 — this is the node that most differs from the original design, so read ADR-0010 before touching it):

**(a) AG review loop**
- **Trigger**: Findings/severity sign-off (checkpoint (b) in ADR-0003) on `findings[]`, followed by Findings Author drafting `transmittal_letter_draft` + `pre_exit_deck_draft` from the signed-off data.
- **State transitions**: `drafted → [Pre-AG Sign-off] → awaiting_ag → changes_requested → ag_feedback_reviewed → (back to Findings Author for redraft) → drafted → [Pre-AG Sign-off] → awaiting_ag → approved`. `ag_status` stays `drafted` through both the findings-data sign-off and the letter/deck drafting — it only advances to `awaiting_ag` once Pre-AG Sign-off resolves, on every pass through the loop, redraft or not.
- **New checkpoint: Pre-AG Sign-off** (**resolves Principal Engineer audit 2.1** — ADR-0003 names three mandatory checkpoints, and this is checkpoint (c), "before anything is submitted for Assistant Governor approval." It is distinct from Findings/severity sign-off: that one approves `findings[]` data before the letter is even drafted; this one is a Lead/Approver review of the **drafted transmittal letter/pre-exit deck itself**, immediately before it reaches the AG. Drawn as `pre_ag_signoff` in `lfi-workflow-phase3-postexam-v1.html`. Raised by Findings Author once both drafts exist; resolved by Lead/Approver; no separate SLA beyond the existing findings/severity sign-off's — see the HITL table.
- **Outputs**: `ag_status`. The actual AG showcase meeting happens outside the pipeline (human process); this agent tracks state and, on `changes_requested`, produces a summarized redraft-notes artifact.
- **No LLM call required for the state machine itself** — this is a status tracker with a human-reported outcome (approved / changes requested), logged to the audit trail.
- **Guardrail (ADR-0017 — fixes Critical Finding #1)**: an LLM may assist in summarizing AG feedback into redraft notes, but that summary is **never passed directly to Findings Author as a redraft trigger**. It surfaces first as its own reviewable artifact behind its own `interrupt()` — "AG Feedback Review" in `lfi-workflow-phase3-postexam-v1.html` — and only Lead/Approver confirmation of that summary advances the state and triggers the redraft. This closes the gap where an ungrounded AI-summarized instruction could otherwise change a regulatory document based on something no human actually said.
- **Guardrail, extended (resolves Principal Engineer audit 2.3)**: a redraft is not exempt from either sign-off. It re-enters at `draft_findings`, re-evaluates severity against the rubric version current at redraft time (ADR-0020), and must pass **both** Findings/severity sign-off and Pre-AG Sign-off again before `ag_status` can reach `awaiting_ag` a second time — the loop-back edges in `lfi-workflow-phase3-postexam-v1.html` route through the full sign-off path, not directly back to `awaiting_ag`. If `rubric_version` differs from the original draft's, that mismatch is surfaced explicitly at the re-run Findings/severity sign-off, per ADR-0020.

**(b) Pre-exit loop**
- **Trigger**: `ag_status == approved`.
- **State transitions**: `scheduled → concerns_raised → (letter/deck updated by Findings Author) → resolved → proceeded_to_exit`, or `scheduled → proceeded_to_exit` directly if the LFI raises no concerns. A `concerns_raised` update reuses the exact same redraft path as (a) — back to Findings Author, then Findings/severity sign-off, then Pre-AG Sign-off — not a shortcut back to `resolved` (decision 105; also resolves the "pre-exit loop drawn only in prose" half of Principal Engineer audit 2.2).
- **Outputs**: `preexit_status`, and — when concerns are raised — the updated letter/deck becomes the **exit deck** (CONTEXT.md).

---

## Cross-cutting

### Examiner Dashboard
Not an agent — the single UI surface (ADR-0008) where every human checkpoint above actually happens. Full screen-by-screen structure settled in ADR-0029; summary:

- **Portfolio home** (landing page): all active examinations — LFI, license type, current phase, SLA countdown/breach flag, open-checkpoint count, last-activity timestamp — filterable Active/Completed/All (default Active). An examination auto-moves Active → Completed the instant `preexit_status` reaches `proceeded_to_exit`; no manual close action.
- **Per-examination workspace**: a permanent sidebar of tabs — **Setup → Intake → Gap Analysis → Meeting → Findings → Approval → Audit Log** — unlocked by data availability, not a locked phase-stepper. Setup is a one-time screen (editable after creation, Lead/Approver only, logged as a distinct audit event) where an LFI is selected, a start date and SFTP root path are set, and the RFI already sent to the LFI is uploaded (auto-parsed into the question list, examiner-confirmed) — the app never generates or sends an RFI itself, and never communicates with the LFI directly at any point (further-submission requests are relayed by the examiner manually, same as the original RFI).
- **RBAC: one screen per tab, gated actions, not parallel UIs.** Examiner and Lead/Approver see identical data/layout in every tab; sign-off controls (approve/edit/reject) render only for Lead/Approver (the only role that can resolve an `interrupt()` — decision 35, reaffirmed in ADR-0029). Examiner and Lead/Approver reach the Audit Log tab only within an examination they're assigned to. A third role, **Auditor/Compliance Reviewer** (ADR-0024), has read-only access to `audit_log[]` and Langfuse traces (ADR-0016); its landing page is a portfolio-wide, cross-examination Audit Log view rather than the per-examination workspace, since it holds no drafting/sign-off authority. It cannot trigger agents or resolve an `interrupt()`, and its access is subject to periodic review.
- **Intake tab**: per-question status (`submitted`/`pending`/`suspicious`), manual refresh, document access via download/open through the Ingestion Adapter (no in-app viewer), and a Lead/Approver-only manual override to force a status to `submitted` with a required reason (logged as a distinct event).
- **Gap Analysis tab**: the `insufficient_grounding`/`needs_human_review` queue; Lead/Approver resolves each item by (a) accepting it as `insufficient_grounding` (becomes a supervision question) or (b) manually asserting a verdict with their own citation (logged human-asserted). Targeted re-run of Compliance Analyst on a single item is explicitly not supported in v1. "Examiner-reviewed" (this section's trigger for Meeting Facilitator, below) is inferred automatically once every flagged item has a resolution and every RFI question has a verdict — no separate sign-off action.
- **Meeting tab**: pre-meeting supervision-question sign-off is per-question (accept/edit/reject each); meeting minutes are submitted as one-shot text after the meeting (no live/real-time capture); post-meeting further-submission requests get their own per-item review checkpoint (see HITL table) before being shown to the examiner to relay to the LFI manually; the sufficiency check is a Lead/Approver judgment call, per agent-specs above.
- **Findings tab**: per-finding accept/edit/reject, qualitative and quantitative tracks shown in visually separate sections (never merged, per ADR-0009/0011). Transmittal letter and pre-exit deck drafts are download-only in v1 (in-browser rendering is a deferred v2 candidate) — the Findings/severity sign-off checkpoint here covers `findings[]` data, not the rendered document; the rendered letter/deck gets its own Pre-AG Sign-off checkpoint (below), which download-only doesn't exempt it from.
- **Approval tab**: hosts both the Pre-AG Sign-off checkpoint (ADR-0003 checkpoint (c) — a Lead/Approver reviews the drafted transmittal letter/pre-exit deck itself, immediately before `ag_status` advances to `awaiting_ag`; resolves Principal Engineer audit 2.1) and AG status tracking, set via dropdown, with required free-text on `changes_requested` capturing what the AG actually said (the input to the LLM redraft-notes summary). The AG Feedback Review checkpoint shows that raw text and the AI summary side-by-side, with only the summary editable, and a confirmed redraft re-enters at Findings Author, then re-passes **both** Findings/severity sign-off and Pre-AG Sign-off before reaching the AG again (resolves audit 2.3 — a redraft can no longer change severity/deadline and reach the AG unreviewed). The pre-exit concerns loop reuses this same full redraft/sign-off flow rather than a separate "Exit Deck" screen — the exit deck is the same artifact, relabeled once produced through this loop.
- **Settings/Admin area** (outside any single examination): the severity/deadline rubric editor — global, per license type, Lead/Approver only, a simple current-state edit form (shows current version, save creates a new version; no browsable version-history UI — full history stays queryable via Audit Log) — and the LFI registry (name, license type, SFTP root, contacts; this app owns this data directly, no external system-of-record integration exists to consume instead).
- **Notifications**: in-app only in v1 (portfolio home's counts/badges); no email. Sustained tool failure (e.g. 3 consecutive missed Intake Tracker polls) shows a persistent "sync degraded" banner — transient retries within ADR-0018's backoff window stay silent.
- **Authentication**: assumed existing CBUAE SSO (session + role claim consumed, no local user/role management screen); specific IdP is an open item, same treatment as `secrets_manager` (ADR-0019).

### Notice Corpus Manager
**Persona**: none — a UI-driven ingestion workflow, not an LLM agent with ongoing state, though it uses an LLM/OCR step internally.

- **Scope**: brought fully in-app for v1 (ADR-0029, decision 108) — no external system or ops process maintains the corpus.
- **Flow**: Lead/Approver uploads a notice (PDF/image) from the Settings/Admin area → auto-OCR (for scanned notices, decision 6) → auto-segmentation into candidate clauses → shown for human review/correction → confirmed clauses become citable by `notice_corpus.search()`/`get_clause()`. No clause is queryable by Compliance Analyst until explicitly confirmed (ADR-0004's zero-tolerance grounding rule applies here too).
- **Supersession review stays reactive**, inline within Gap Analysis's `needs_human_review` path (agent-specs above) — no standalone corpus-wide supersession-graph browser in v1.
- **Access**: Lead/Approver only; no separate Compliance/Legal Content Owner role introduced.
- **Open item**: scoped as a direction, no tool contract yet (unlike the sanctions-screening gap in ADR-0028, which at least has real MCP servers to point to).

### Audit Trail Store
Not an agent — every agent above writes an entry here (via a shared `log_checkpoint()` call, full contract in `tool-contracts.md`) at every `interrupt()` resolution and every state-machine transition in the AG/Pre-Exit Agent. Append-only (decision 36); physically part of the Fraud Database (ADR-0012). Entries are hash-chained (each includes a hash of the prior entry) so tampering is detectable (ADR-0016).

**`AuditLogEntry` schema** (Principal Engineer audit 3.2/6.2 — previously `audit_log[]` had no field-level schema anywhere, despite six load-bearing requirements on it scattered across this document and several ADRs; a hash chain cannot be retrofitted onto an existing log, so this had to be specified before any DB implementation starts, not after):

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | primary key |
| `examination_id` | string | which examination's chain this entry belongs to — the hash chain is per-examination, not global |
| `timestamp` | ISO 8601 | |
| `actor` | `{ type: "agent"\|"human", name: string, user_id?: string }` | `name` is the agent persona (e.g. `"Compliance Analyst"`) or the resolving Lead/Approver's identity |
| `event_type` | namespaced string, e.g. `"checkpoint.resolved"`, `"intake.override"`, `"intake.resubmission_requested"`, `"setup.corrected"`, `"gap_analysis.human_asserted_verdict"`, `"gap_analysis.insufficient_grounding"`, `"gap_analysis.low_confidence_supersession"`, `"examination.voided"`, `"findings.rubric_version_changed_on_redraft"` | an open, growing namespace by design (new checkpoint/override types will keep adding values) rather than a closed enum that goes stale; every place in this document that requires "a distinct event type" or "logged distinctly" names the specific `event_type` value it needs |
| `state_before`, `state_after` | object | scoped to the fields the triggering action actually touched, not a full state snapshot |
| `resolution` | `"accept"\|"edit"\|"reject"` (optional) | present only on `checkpoint.*` entries |
| `edit_diff` | object (JSON Patch), optional | present only when `resolution: "edit"` — this is what makes the human-agreement-rate metric (ADR-0016) measurable, not just "was it resolved" |
| `reason_code` | string, optional | e.g. `"insufficient_grounding"` vs. `"low_confidence_supersession"` (decision 30) — carries the "distinct reasons, same queue" requirement from Compliance Analyst's Human Review Flags |
| `idempotency_key` | string | deterministic hash of `(examination_id, langgraph_node_execution_id)` — computed by `log_checkpoint()` itself from the LangGraph execution context, never supplied by the caller, so a replayed/resumed node can't accidentally pass a fresh one (ADR-0018) |
| `prev_hash` | string | hash of the immediately preceding entry in this `examination_id`'s chain; the first entry per examination uses a fixed genesis seed |
| `entry_hash` | string | hash over this entry's other fields concatenated with `prev_hash` — the tamper-evidence mechanism ADR-0016 requires |

---

## HITL checkpoints

Every checkpoint below is a LangGraph `interrupt()` that only a Lead/Approver can resolve (decision 35). `log_checkpoint()` records the agent's proposed output, the human's action (accept / edit / reject), and any edit diff — this is what makes human-agreement rate measurable (ADR-0016), not just "was it resolved."

| Checkpoint | Raised by | Resolved by | What's being approved | Suggested SLA |
|---|---|---|---|---|
| Gap-analysis review | Compliance Analyst | Lead/Approver | `insufficient_grounding` / `needs_human_review` items before supervision questions are finalized | 2 business days |
| Supervision-question sign-off | Meeting Facilitator | Lead/Approver | Final question list before the live meeting | Before the scheduled meeting (hard deadline, not a duration) |
| Further-submission review *(added — ADR-0029, decision 99)* | Meeting Facilitator | Lead/Approver | `further_submission_requests[]` extracted from meeting minutes, before the list is final and relayed to the LFI | Not yet set — tune empirically, same treatment as decision 30/54 |
| Sufficiency check *(added — Critical Finding #3)* | Meeting Facilitator | Lead/Approver | Whether further-submission responses actually close the meeting's gaps | 2 business days from further-submission receipt |
| Findings/severity sign-off | Findings Author | Lead/Approver | Findings + severity before transmittal letter drafting | 3 business days |
| Pre-AG sign-off *(added — ADR-0003 checkpoint (c), Principal Engineer audit 2.1)* | Findings Author | Lead/Approver | The drafted transmittal letter/pre-exit deck itself, before ag_status advances to awaiting_ag | 2 business days |
| AG review loop | Approval Coordinator | Assistant Governor (outcome relayed by Lead/Approver) | Transmittal letter + pre-exit deck | No pipeline-enforced SLA — external process; track time-in-state for visibility only |
| AG feedback review *(added — Critical Finding #1)* | Approval Coordinator | Lead/Approver | The LLM-summarized AG feedback itself, before it can trigger a redraft | 1 business day (blocks the redraft cycle) |
| Pre-exit concerns loop | Approval Coordinator | Lead/Approver | Updated letter/deck after LFI raises concerns | 2 business days from concerns raised |

SLA breaches and current backlog (count of unresolved checkpoints per type) surface on the Examiner Dashboard — this is what a Lead/Approver should see first, not something they have to query for.

---

## Definition of done, per agent (ADR-0026)

Today "done" is implicit — a guardrail passed and a human accepted. This section makes each agent's completion condition an explicit, checkable predicate, distilled from guardrails/state fields already defined above rather than a new mechanism:

- **Intake Tracker**: every RFI question resolves to a terminal `intake_status` of `submitted` or `pending` (`suspicious` is transient — see Intake Tracker's own section above for its two resolution paths); `all_submitted` fires only when 100% are `submitted`.
- **Compliance Analyst**: every RFI question has exactly one `compliance_verdicts[]` entry that is either (a) grounded with a citation machine-verifiably present in Notice Corpus, or (b) explicitly `insufficient_grounding`/`needs_human_review` — zero questions with no verdict at all, and no verdict skips the supersession check.
- **Meeting Facilitator**: pre-meeting — every open `supervision_questions[]` item is non-empty, cites a specific gap, and is Lead/Approver-signed-off before the meeting's hard deadline. Post-meeting — every `further_submission_requests[]` item traces to a specific unresolved question, and the sufficiency gate has produced an explicit `sufficient`/`insufficient` — `meeting_followup_status` is never left unset.
- **Findings Author**: every `findings[]` entry has `severity` equal to the last `severity_rubric.lookup()` return for its `rubric_version` (ADR-0020, code-enforced), every quantitative claim traces to a `quant_analysis.compute()` call (ADR-0021, code-enforced), and the transmittal letter/pre-exit deck drafts are non-empty and reference every signed-off finding.
- **Approval Coordinator**: `ag_status`/`preexit_status` always lands on a named terminal or waiting state, never null/undefined; AG-summarized feedback is never treated as processed until the Lead/Approver has explicitly confirmed the "AG Feedback Review" checkpoint (ADR-0017); `ag_status` never reaches `awaiting_ag` without a Pre-AG Sign-off resolution logged for the draft currently in flight, redraft or not (ADR-0003 checkpoint (c)).

No agent's output is "verified" by a second LLM pass — see ADR-0026 for why a dedicated Verifier agent was considered and rejected in favor of these code-checkable predicates plus the existing HITL checkpoints.

---

## Failure, retry, and idempotency

Full rationale: ADR-0018. Four policies apply uniformly across every agent and every checkpoint above — this section is the concrete "what to build" version, not a repeat of the ADR's reasoning.

1. **Tool-call retries.** Every tool call in this document (`notice_corpus.*`, `edm.query`, `supersession_graph.check`, `severity_rubric.lookup`, `template_fill`, `ingestion_adapter.*`) retries transient failures with bounded backoff. A tool-unavailable outcome is a distinct code path from "no result found" — never collapse them, especially for `notice_corpus.get_clause()` under the citation-grounding rule (ADR-0004).
2. **Idempotency keys.** Every state-mutating agent invocation and every `log_checkpoint()` call carries an idempotency key, so a replayed or re-run LangGraph node never double-writes state or double-logs an `audit_log[]` entry.
3. **Optimistic concurrency on checkpoints.** Every `interrupt()` carries a version/etag. A second concurrent resolution attempt on an already-resolved checkpoint is rejected with an explicit conflict — never silently overwritten, never double-applied.
4. **Structured-output validation and repair.** Every agent's output is schema-validated before being accepted into state. One automatic repair retry is allowed (re-prompt with the validation error attached); a second failure escalates to human review rather than accepting malformed output or silently coercing it.

---

## Observability and evaluation

Full stack rationale: ADR-0016. This section is the concrete metric contract per agent — what to emit, not just where it goes.

### Golden signals (every agent and service)
Rate, Errors, Duration (RED method) via OpenTelemetry: invocation count, error rate (including `insufficient_grounding` and other refusal outcomes — these are not errors, tag them separately from actual failures), p50/p95/p99 latency. The LLM Inference Service additionally reports GPU utilization, queue depth, and saturation (USE method).

### Agent-quality (evaluation) metrics

| Metric | Agent | What it catches |
|---|---|---|
| Citation-validity rate | Compliance Analyst | % of verdicts whose cited clause ID machine-verifiably exists in Notice Corpus and matches the quoted text — automatable, gate CI/regression tests on this before any model or prompt change ships |
| Insufficient-grounding rate | Compliance Analyst | Trending up = corpus gap or model regression; trending down over time = corpus maturing. A sudden spike after a model swap (decision 49 — model stays swappable) is the first thing to check |
| Human-agreement rate | All (per checkpoint in the HITL table) | **The primary trust metric.** Falling = quality regression. Near-100% = investigate for automation bias (decision 1's augmentation model depends on real review happening) |
| Override/edit rate, by reason if captured | All | Inverse of agreement rate; bucket by reason to find systematic weak spots (e.g., always over-escalating one clause type) |
| Supersession auto-resolve vs. escalate ratio | Compliance Analyst | Tunes the confidence threshold (decision 30) against real cases rather than guessing |
| Revision-loop iteration count | Approval Coordinator | Proxy for draft quality — repeated `changes_requested` cycles on the same letter is a signal, not just noise |

### How evaluation actually runs
Offline: a golden eval set (real, anonymized past examinations) re-run in CI before any prompt or model change ships — citation-validity and a small human-graded sample are the release gate. Online: production traces sampled into Langfuse for human spot-check scoring, plus the always-on automated citation-verifier. Both write scores to the same trace, so a regression can be traced back to a specific prompt/model version.

---

## Testing, canary, and rollback (ADR-0023)

The golden eval set above gates citation quality; it catches nothing about the graph's structural correctness or a mid-cycle model swap's blast radius. Required before implementation is considered done:
1. **Unit tests** for every tool contract's error/timeout path (`docs/architecture/tool-contracts.md`) — not just the happy path.
2. **LangGraph integration tests** exercising every `interrupt()`/resume path and both AG/pre-exit revision loops (ADR-0010) against a simulated crash-and-resume.
3. **Canary process** for any model/prompt swap (decision 49 keeps the model swappable): candidate runs against the golden eval set plus a shadow-traffic sample before promotion, with a pilot-subset human-agreement-rate comparison before full cutover.
4. **In-flight examinations are pinned** to the model version they started under until a natural phase boundary (end of Phase 1/2, or completion) — never silently switched mid-clause-verdict-loop.

## Capacity (ADR-0022, placeholder)

No real FPSD usage data exists yet — treat every number here as illustrative, revisit once the examiner team's actual annual examination count and concurrency are known (same "tune empirically" treatment as decision 30's supersession threshold). Rough placeholder: ~8 peak-concurrent examinations, ~40 RFI questions each driving Compliance Analyst's per-clause ReAct loop, pointing to a 2-4 GPU placeholder for the LLM Inference Service — moves once the model choice (decision 49: Qwen Coder vs. gpt-oss-120b, neither committed) and real concurrency are known. Full methodology in ADR-0022.

## Threat model and network isolation (ADR-0019)

The LLM Inference Service sits in its own `security-group` boundary in the deployment diagram — only the four LLM agents (not Intake Tracker, which is rule-based) may call it, over mTLS. The Fraud Database uses encryption at rest (TDE or equivalent). A `secrets_manager` component holds SFTP credentials, the Oracle connection string, and service-auth tokens — which specific product is an open item for CBUAE's platform team, not assumed here. Full rationale in ADR-0019; a fuller threat-modeling exercise (ingestion-path attack surface, formal STRIDE pass, CBUAE security sign-off) remains explicitly out of scope for this document.

---

## What's deliberately not specified here

Per ADR-0002/`docs/adr/0002`, these are out of v1 and have no agent contract yet: content/plausibility validation (a document-opening judgment agent), a unified cross-feature review queue, effective-dated notice versioning, and anything for SVF/other license types. Do not build stubs for these — add their specs when they're actually scoped.

Also deferred, per later decisions: a dedicated Verifier agent (ADR-0026 — considered and rejected; code guardrails + HITL checkpoints do this job instead) and any external-intelligence enrichment beyond official sanctions/watchlist screening (ADR-0028 — adverse-media/news/consumer-complaint monitoring is on hold pending CBUAE legal/compliance sign-off on whether FPSD is authorized to build it at all; sanctions screening against official government lists is the one piece scoped as a near-term addition, but no tool contract exists for it yet).
