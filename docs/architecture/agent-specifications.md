# Agent Specifications — LFI Examination Pipeline v1

Detailed contract for every agentic and non-agentic component in `docs/architecture/lfi-pipeline-v1.html` and `lfi-pipeline-deployment-v1.html`. Each agent is a LangGraph node (or small subgraph); this document is the source of truth for what to actually build — inputs, outputs, tools, model behavior, state, human checkpoints (with SLAs), and the observability/evaluation contract each agent must satisfy. Read `CONTEXT.md` for vocabulary and `docs/adr/` for the decisions behind these contracts before changing them.

**The actual LangGraph node/edge/conditional-transition structure — not just this prose — lives in three workflow diagrams**, one per phase: `lfi-workflow-phase1-preexam.html`, `lfi-workflow-phase2-meeting.html`, `lfi-workflow-phase3-postexam.html`. Read the relevant one before implementing a phase; it shows every guardrail branch, human checkpoint, and (Phase 2) the sufficiency-check gate that this document's prose alone does not make unambiguous.

**Every tool named below has a real input/output/timeout/retry contract in `docs/architecture/tool-contracts.md`** — this document names which tool each agent calls and why; that one specifies exactly what calling it looks like, including its distinct error/timeout shape (never collapsed into "no result found," ADR-0018).

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
| `meeting_followup_status` (sufficient / insufficient) | Clarification & Meeting Agent, via Lead/Approver sufficiency check | Findings & Reporting Agent (its actual trigger condition), Examiner Dashboard |
| `findings[]` (severity, deadline, `rubric_version`, citations) | Findings & Reporting Agent | Examiner Dashboard, AG & Pre-Exit Agent |
| `transmittal_letter_draft`, `pre_exit_deck_draft` | Findings & Reporting Agent | AG & Pre-Exit Agent |
| `ag_status` (drafted / awaiting_ag / changes_requested / ag_feedback_reviewed / approved) | AG & Pre-Exit Agent | Examiner Dashboard |
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
  - **Every tool call above retries transient failures (timeout, connection error) with backoff (ADR-0018) — a tool-unavailable result is never treated as "no matching clause/data," which would silently misfire the grounding rule below.**
- **Model behavior contract (hard rules, not prompted preferences — enforce in code, not just the prompt)**:
  1. **No verdict without a grounded citation.** If `notice_corpus.get_clause()` returns no high-confidence exact match (a real absence, not a timeout — see ADR-0018), the verdict is `insufficient_grounding` — never a best-guess compliant/non-compliant call (ADR-0004).
  2. Supersession confidence below the configured threshold (decision 30 — tune empirically; start conservative) routes that clause pair to `needs_human_review` rather than resolving it silently.
  3. In the workflow diagram (`lfi-workflow-phase1-preexam.html`), both escalation reasons above are drawn as one merged "Human Review Flags" state — they route to the same Examiner Dashboard queue but must still be logged as distinct reasons in `audit_log[]`.
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

**(c) Sufficiency check — added after architecture review (Critical Finding #3)**
`current_state.png`'s Phase 2 shows an explicit, iterative "documents/answers sufficient?" loop that earlier drafts of this spec left unmodeled, treating post-meeting extraction as one-shot with an undefined "further-submission loop resolved" condition. This is now a real, named gate (`lfi-workflow-phase2-meeting.html`'s "Sufficiency Check" node):
- **Trigger**: `further_submission_requests[]` outcomes received back through Intake Tracker's next cycle.
- **Decision maker**: **Lead/Approver, not the LLM** (ADR-0003's augmentation principle — this is a judgment call about whether the LFI's response actually closes the gap, not an extraction task).
- **Outputs**: `meeting_followup_status` (`sufficient` / `insufficient`). `insufficient` re-enters Phase 1's Intake Tracker via the same `all_submitted` trigger for another document-collection round; `sufficient` is the condition that actually satisfies Findings & Reporting Agent's trigger below — "further-submission loop resolved" now means exactly this state value, not a prose assumption.

### Findings & Reporting Agent
**Persona**: **Findings Author** — writes for a reader (the Assistant Governor, then LFI leadership) who wasn't in the room: every finding stands on its own, with severity and citation, no institutional memory assumed.

- **Trigger**: `meeting_followup_status == sufficient` (the Clarification & Meeting Agent's sufficiency check, above — not an assumed "resolved").
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

### AG & Pre-Exit Agent
**Persona**: **Approval Coordinator** — a state-tracker, not a persuader; it never drafts arguments to win over the AG or the LFI, it only tracks what was asked for and routes redrafts. Where an LLM is involved (summarizing feedback into redraft notes), it stays strictly descriptive.

Two loops, both routine state (ADR-0010 — this is the node that most differs from the original design, so read ADR-0010 before touching it):

**(a) AG review loop**
- **Trigger**: signed-off `transmittal_letter_draft` + `pre_exit_deck_draft`.
- **State transitions**: `drafted → awaiting_ag → changes_requested → ag_feedback_reviewed → (back to Findings & Reporting for redraft) → awaiting_ag → approved`.
- **Outputs**: `ag_status`. The actual AG showcase meeting happens outside the pipeline (human process); this agent tracks state and, on `changes_requested`, produces a summarized redraft-notes artifact.
- **No LLM call required for the state machine itself** — this is a status tracker with a human-reported outcome (approved / changes requested), logged to the audit trail.
- **Guardrail (ADR-0017 — fixes Critical Finding #1)**: an LLM may assist in summarizing AG feedback into redraft notes, but that summary is **never passed directly to Findings & Reporting as a redraft trigger**. It surfaces first as its own reviewable artifact behind its own `interrupt()` — "AG Feedback Review" in `lfi-workflow-phase3-postexam.html` — and only Lead/Approver confirmation of that summary advances the state to `ag_feedback_reviewed` and triggers the redraft. This closes the gap where an ungrounded AI-summarized instruction could otherwise change a regulatory document based on something no human actually said.

**(b) Pre-exit loop**
- **Trigger**: `ag_status == approved`.
- **State transitions**: `scheduled → concerns_raised → (letter/deck updated by Findings & Reporting) → resolved → proceeded_to_exit`, or `scheduled → proceeded_to_exit` directly if the LFI raises no concerns.
- **Outputs**: `preexit_status`, and — when concerns are raised — the updated letter/deck becomes the **exit deck** (CONTEXT.md).

---

## Cross-cutting

### Examiner Dashboard
Not an agent — the single UI surface (ADR-0008) where every human checkpoint above actually happens: reviewing gap-analysis flags, signing off supervision questions, submitting meeting minutes, signing off findings/severity, and recording AG/pre-exit outcomes. RBAC: Examiner (drafts, triggers agents) vs. Lead/Approver (the only role that can resolve an `interrupt()`) — decision 35.

### Audit Trail Store
Not an agent — every agent above writes an entry here (via a shared `log_checkpoint(state_before, state_after, actor)` call) at every `interrupt()` resolution and every state-machine transition in the AG/Pre-Exit Agent. Append-only (decision 36); physically part of the Fraud Database (ADR-0012). Entries are hash-chained (each includes a hash of the prior entry) so tampering is detectable (ADR-0016).

---

## HITL checkpoints

Every checkpoint below is a LangGraph `interrupt()` that only a Lead/Approver can resolve (decision 35). `log_checkpoint()` records the agent's proposed output, the human's action (accept / edit / reject), and any edit diff — this is what makes human-agreement rate measurable (ADR-0016), not just "was it resolved."

| Checkpoint | Raised by | Resolved by | What's being approved | Suggested SLA |
|---|---|---|---|---|
| Gap-analysis review | Gap Analysis Agent | Lead/Approver | `insufficient_grounding` / `needs_human_review` items before supervision questions are finalized | 2 business days |
| Supervision-question sign-off | Clarification & Meeting Agent | Lead/Approver | Final question list before the live meeting | Before the scheduled meeting (hard deadline, not a duration) |
| Sufficiency check *(added — Critical Finding #3)* | Clarification & Meeting Agent | Lead/Approver | Whether further-submission responses actually close the meeting's gaps | 2 business days from further-submission receipt |
| Findings/severity sign-off | Findings & Reporting Agent | Lead/Approver | Findings + severity before transmittal letter drafting | 3 business days |
| AG review loop | AG & Pre-Exit Agent | Assistant Governor (outcome relayed by Lead/Approver) | Transmittal letter + pre-exit deck | No pipeline-enforced SLA — external process; track time-in-state for visibility only |
| AG feedback review *(added — Critical Finding #1)* | AG & Pre-Exit Agent | Lead/Approver | The LLM-summarized AG feedback itself, before it can trigger a redraft | 1 business day (blocks the redraft cycle) |
| Pre-exit concerns loop | AG & Pre-Exit Agent | Lead/Approver | Updated letter/deck after LFI raises concerns | 2 business days from concerns raised |

SLA breaches and current backlog (count of unresolved checkpoints per type) surface on the Examiner Dashboard — this is what a Lead/Approver should see first, not something they have to query for.

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
| Citation-validity rate | Gap Analysis | % of verdicts whose cited clause ID machine-verifiably exists in Notice Corpus and matches the quoted text — automatable, gate CI/regression tests on this before any model or prompt change ships |
| Insufficient-grounding rate | Gap Analysis | Trending up = corpus gap or model regression; trending down over time = corpus maturing. A sudden spike after a model swap (decision 49 — model stays swappable) is the first thing to check |
| Human-agreement rate | All (per checkpoint in the HITL table) | **The primary trust metric.** Falling = quality regression. Near-100% = investigate for automation bias (decision 1's augmentation model depends on real review happening) |
| Override/edit rate, by reason if captured | All | Inverse of agreement rate; bucket by reason to find systematic weak spots (e.g., always over-escalating one clause type) |
| Supersession auto-resolve vs. escalate ratio | Gap Analysis | Tunes the confidence threshold (decision 30) against real cases rather than guessing |
| Revision-loop iteration count | AG & Pre-Exit | Proxy for draft quality — repeated `changes_requested` cycles on the same letter is a signal, not just noise |

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

No real FPSD usage data exists yet — treat every number here as illustrative, revisit once the examiner team's actual annual examination count and concurrency are known (same "tune empirically" treatment as decision 30's supersession threshold). Rough placeholder: ~8 peak-concurrent examinations, ~40 RFI questions each driving Gap Analysis Agent's per-clause ReAct loop, pointing to a 2-4 GPU placeholder for the LLM Inference Service — moves once the model choice (decision 49: Qwen Coder vs. gpt-oss-120b, neither committed) and real concurrency are known. Full methodology in ADR-0022.

## Threat model and network isolation (ADR-0019)

The LLM Inference Service sits in its own `security-group` boundary in the deployment diagram — only the four LLM agents (not Intake Tracker, which is rule-based) may call it, over mTLS. The Fraud Database uses encryption at rest (TDE or equivalent). A `secrets_manager` component holds SFTP credentials, the Oracle connection string, and service-auth tokens — which specific product is an open item for CBUAE's platform team, not assumed here. Full rationale in ADR-0019; a fuller threat-modeling exercise (ingestion-path attack surface, formal STRIDE pass, CBUAE security sign-off) remains explicitly out of scope for this document.

---

## What's deliberately not specified here

Per ADR-0002/`docs/adr/0002`, these are out of v1 and have no agent contract yet: content/plausibility validation (a document-opening judgment agent), a unified cross-feature review queue, effective-dated notice versioning, and anything for SVF/other license types. Do not build stubs for these — add their specs when they're actually scoped.
