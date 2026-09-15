# Agent Specifications — LFI Examination Pipeline v1

Detailed contract for every agentic and non-agentic component in `docs/architecture/lfi-pipeline-v1.html`. Each agent is a LangGraph node (or small subgraph); this document is the source of truth for what to actually build — inputs, outputs, tools, model behavior, state, and human checkpoints. Read `CONTEXT.md` for vocabulary and `docs/adr/` for the decisions behind these contracts before changing them.

## Shared graph state

All agents read and write a single LangGraph state object, scoped per examination (one instance per LFI per examination cycle). Fields referenced below:

| Field | Written by | Read by |
|---|---|---|
| `rfi_responses[]` | Intake Tracker | Gap Analysis Agent |
| `intake_status` (per RFI question: submitted / pending / suspicious) | Intake Tracker | Examiner Dashboard |
| `compliance_verdicts[]` | Gap Analysis Agent | Examiner Dashboard, Clarification & Meeting Agent |
| `supervision_questions[]` | Gap Analysis Agent, Clarification & Meeting Agent | Examiner Dashboard, Audit Trail |
| `meeting_minutes` (raw text) | Examiner Dashboard (human input) | Clarification & Meeting Agent |
| `further_submission_requests[]` | Clarification & Meeting Agent | Intake Tracker (next cycle) |
| `findings[]` (severity, deadline, citations) | Findings & Reporting Agent | Examiner Dashboard, AG & Pre-Exit Agent |
| `transmittal_letter_draft`, `pre_exit_deck_draft` | Findings & Reporting Agent | AG & Pre-Exit Agent |
| `ag_status` (drafted / awaiting_ag / changes_requested / approved) | AG & Pre-Exit Agent | Examiner Dashboard |
| `preexit_status` (scheduled / concerns_raised / resolved / proceeded_to_exit) | AG & Pre-Exit Agent | Examiner Dashboard |
| `audit_log[]` (append-only) | every agent, on every state transition | Audit Trail Store |

State is persisted to the Fraud Database (ADR-0012) after every node execution — this is what makes LangGraph's interrupt/resume (ADR-0001) durable across the 30-day RFI window and the days/weeks between examination phases.

---

## Phase 1 — Pre-Examination

### Intake Tracker
**Persona**: none — **not an LLM agent**, a deterministic rule-based service. Included here because it gates Gap Analysis.

- **Trigger**: scheduled poll of the Ingestion Adapter (e.g., every 15 min) during the 30-day RFI window, and on-demand via Examiner Dashboard refresh.
- **Inputs**: `rfi_store` (expected question → folder/filename convention, per decision 15), Ingestion Adapter's `list_documents()` output.
- **Outputs**: `intake_status` per RFI question — `submitted` (present, non-empty, filename/format matches), `pending` (absent), or `suspicious` (present but fails the format check — decision 18). Deferred to v2: any content/plausibility judgment (decision 28) — this service never opens a file to judge whether it's a *plausible* answer.
- **Tools**: `ingestion_adapter.list_documents(lfi_id)`, `ingestion_adapter.get_metadata(doc_id)` (size, filename, last-modified).
- **Trigger for Gap Analysis**: fires the `all_submitted` event only when every RFI question's status is `submitted` (decision 39 — batch, not incremental).

### Gap Analysis Agent
**Persona**: **Compliance Analyst** — meticulous, cites chapter and verse, and says "I can't confirm this" rather than guess. System-prompt framing should make refusal-to-guess feel like professional competence, not failure, since the grounding rule (below) depends on the model being comfortable abstaining.

- **Trigger**: `all_submitted` event from Intake Tracker.
- **Inputs**: `rfi_responses[]` (documents), Notice Corpus (clause citations), EDM (fraud-data context, decision 21).
- **Outputs**: `compliance_verdicts[]` and the three Phase-1 artifacts — **supervision questions**, **LFI gaps and findings**, **areas of strengths and weaknesses** (decision 67 in `CONTEXT.md`'s Gap Analysis entry).
- **Tools**:
  - `notice_corpus.search(query, clause_filter)` — semantic + exact-match retrieval over the per-clause index.
  - `notice_corpus.get_clause(clause_id)` — exact verbatim sentence lookup, required before any verdict (ADR-0004).
  - `edm.query(lfi_id, period)` — read-only, context only, never written back.
  - `supersession_graph.check(clause_id)` — returns superseding clause + confidence score (decision 5, 30).
- **Model behavior contract (hard rules, not prompted preferences — enforce in code, not just the prompt)**:
  1. **No verdict without a grounded citation.** If `notice_corpus.get_clause()` returns no high-confidence exact match, the verdict is `insufficient_grounding` — never a best-guess compliant/non-compliant call (ADR-0004).
  2. Supersession confidence below the configured threshold (decision 30 — tune empirically; start conservative) routes that clause pair to `needs_human_review` rather than resolving it silently.
- **LangGraph shape**: a ReAct-style subgraph (retrieve → verify citation → verdict) looped once per RFI question/clause pair, not a single call over the whole document set — keeps each verdict independently inspectable and re-runnable.
- **Human checkpoint**: none at this node itself; `insufficient_grounding` and `needs_human_review` items surface in the Examiner Dashboard's gap-analysis review screen for the examiner to resolve before supervision questions are finalized.

---

## Phase 2 — Examination (Onsite)

### Clarification & Meeting Agent
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

### Findings & Reporting Agent
**Persona**: **Findings Author** — writes for a reader (the Assistant Governor, then LFI leadership) who wasn't in the room: every finding stands on its own, with severity and citation, no institutional memory assumed.

- **Trigger**: meeting complete, further-submission loop (if any) resolved.
- **Inputs**: `compliance_verdicts[]`, `further_submission_requests[]` outcomes, EDM (for quantitative track).
- **Outputs**: `findings[]` split into two parallel analysis tracks (ADR-0011):
  - **Qualitative**: notice/clause compliance narrative — carries forward each finding's grounding citation.
  - **Quantitative**: EDM figures analyzed on their own terms (loss ratios, volume trends) — **never** reconciled against document claims (ADR-0009/0011 — this is the one rule most likely to be broken by a well-intentioned future engineer; do not add a "compare to document" tool here).
  - `transmittal_letter_draft` and `pre_exit_deck_draft`, filled into the fixed Word/PPT templates (decision 9) from `findings[]` — the deck is a derived artifact off the same data, not independently authored (decision 26).
- **Tools**: `severity_rubric.lookup(finding)` → severity + deadline (decision 16 — rubric is data, UI-editable, not hardcoded in this agent), `template_fill(template_id, findings[])`.
- **Human checkpoint**: **required sign-off** on findings/severity before the transmittal letter is drafted (decision 1, decision 26). `interrupt()` here.

### AG & Pre-Exit Agent
**Persona**: **Approval Coordinator** — a state-tracker, not a persuader; it never drafts arguments to win over the AG or the LFI, it only tracks what was asked for and routes redrafts. Where an LLM is involved (summarizing feedback into redraft notes), it stays strictly descriptive.

Two loops, both routine state (ADR-0010 — this is the node that most differs from the original design, so read ADR-0010 before touching it):

**(a) AG review loop**
- **Trigger**: signed-off `transmittal_letter_draft` + `pre_exit_deck_draft`.
- **State transitions**: `drafted → awaiting_ag → changes_requested → (back to Findings & Reporting for redraft) → awaiting_ag → approved`.
- **Outputs**: `ag_status`. The actual AG showcase meeting happens outside the pipeline (human process); this agent only tracks state and routes a `changes_requested` result back into Findings & Reporting as a redraft trigger.
- **No LLM call required for the state machine itself** — this is a status tracker with a human-reported outcome (approved / changes requested), logged to the audit trail. An LLM may assist in summarizing AG feedback into actionable redraft notes, but the state transition itself is deterministic.

**(b) Pre-exit loop**
- **Trigger**: `ag_status == approved`.
- **State transitions**: `scheduled → concerns_raised → (letter/deck updated by Findings & Reporting) → resolved → proceeded_to_exit`, or `scheduled → proceeded_to_exit` directly if the LFI raises no concerns.
- **Outputs**: `preexit_status`, and — when concerns are raised — the updated letter/deck becomes the **exit deck** (CONTEXT.md).

---

## Cross-cutting

### Examiner Dashboard
Not an agent — the single UI surface (ADR-0008) where every human checkpoint above actually happens: reviewing gap-analysis flags, signing off supervision questions, submitting meeting minutes, signing off findings/severity, and recording AG/pre-exit outcomes. RBAC: Examiner (drafts, triggers agents) vs. Lead/Approver (the only role that can resolve an `interrupt()`) — decision 35.

### Audit Trail Store
Not an agent — every agent above writes an entry here (via a shared `log_checkpoint(state_before, state_after, actor)` call) at every `interrupt()` resolution and every state-machine transition in the AG/Pre-Exit Agent. Append-only (decision 36); physically part of the Fraud Database (ADR-0012).

---

## What's deliberately not specified here

Per ADR-0002/`docs/adr/0002`, these are out of v1 and have no agent contract yet: content/plausibility validation (a document-opening judgment agent), a unified cross-feature review queue, effective-dated notice versioning, and anything for SVF/other license types. Do not build stubs for these — add their specs when they're actually scoped.
