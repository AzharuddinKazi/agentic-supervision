# Principal Engineer Design Review — LFI Examination Pipeline + Examiner Dashboard

**Reviewer:** Principal Engineer (independent, third review pass)
**Date:** 2026-09-17
**Scope reviewed:** `CONTEXT.md`; all 29 ADRs (`docs/adr/`); `docs/architecture/agent-specifications.md`; `docs/architecture/tool-contracts.md`; all 19 `docs/architecture/*.candidate.json` diagrams (verified against directory listing — 19 present, matching the brief's list exactly, no additions). `docs/decision-log.md` consulted for decision provenance only.
**Method:** MECE audit across 6 dimensions. Prior reviews in `.scratch/` were deliberately not opened until this document was drafted; cross-check section at the end.

**Headline:** this is a genuinely strong design corpus — above the median of what I see at design review, with an unusually disciplined deferral log. The weaknesses are not in the thinking; they are in the **build-level substrate** (state schema, tool contracts, audit-record schema) failing to keep up with how far the diagrams have run ahead, and in **three mandatory checkpoints that exist in one artifact and are missing from another**.

---

## Dimension 1 — Architecture correctness & completeness

Scope: `lfi-pipeline-context-v1`, `lfi-pipeline-v1`, `lfi-pipeline-deployment-v1`.

### 1.1 — The Ingestion Adapter does not exist in the deployment diagram; SFTP is drawn straight into Intake Tracker — **High**
`lfi-pipeline-v1.candidate.json` models `lfi_workspace → ingestion_adapter → intake_tracker` and ADR-0006 makes the adapter a first-class, deliberately-abstracted boundary ("without the adapter boundary, the smart-portal migration would require touching every downstream consumer"). `lfi-pipeline-deployment-v1.candidate.json` has **no `ingestion_adapter` component at all** — connection `workspace-to-intake` runs `lfi_workspace → intake_tracker` labelled "SFTP", and `secrets-to-intake` delivers "SFTP creds" to `intake_tracker`, while ADR-0019 §3 says the secrets manager "feed[s] the Ingestion Adapter (SFTP creds)".
Two competent engineers read these differently: one builds a separately-deployed adapter service with the ADR-0006 interface; the other builds SFTP directly into the Intake Tracker process and the adapter abstraction quietly dies. That is precisely the outcome ADR-0006 was written to prevent.

### 1.2 — Three v1 platform services with full tool contracts have no home in any architecture diagram — **High**
`severity_rubric` (ADR-0020 — explicitly a *shared platform service*, own release cycle, 3s latency budget), `quant_analysis.compute` (ADR-0021 — deterministic compute service), and `template_fill` (Word/PPT generation, 15s timeout) all have first-class contracts in `tool-contracts.md` but appear as components in **neither** `lfi-pipeline-v1` nor `lfi-pipeline-deployment-v1`. The rubric appears only as a node inside `lfi-rubric-editor-v1`. Nobody owns them on the deployment diagram (the diagram's `engineering_profile` is literally `deployment-ownership` and every other box carries a team `tag`). Document generation in particular is a real service with template storage, and `template_fill` returns a `document_ref` "into wherever drafted documents are stored" — a store that appears in no diagram.

### 1.3 — Notice Corpus Manager is absent from the deployment diagram and mis-typed in the logical one — **Medium**
`lfi-pipeline-v1` types `notice_corpus_mgr` as `messagebus` = "LLM Agent + Tool Call". `agent-specifications.md` §Notice Corpus Manager says "**Persona**: none — a UI-driven ingestion workflow, **not an LLM agent**". `lfi-notice-corpus-manager-sequence-v1` shows only a `cloud`-typed "OCR + Segmenter — deterministic tool" and no LLM participant at all. Three artifacts, three different answers to "is there an LLM in this path?" — which matters, because if there is, it inherits ADR-0019's mTLS inference-zone rule and ADR-0016's Langfuse tracing, and if there isn't, it doesn't. It is also missing entirely from the deployment diagram despite being in-app v1 scope (ADR-0029).

### 1.4 — Only one representative edge drawn for LLM inference, DB persistence, and observability, with no "representative, not exhaustive" annotation — **Medium**
Deployment draws `clarification_agent → llm_inference` only (ADR-0019 requires **four** agents to hold mTLS routes into that zone), `findings_reporting → oracle_db` only (`agent-specifications.md`: state "is persisted to the Fraud Database after **every node execution**"), and `ag_preexit → observability` only (ADR-0016: "**every** agent and service emits OTel"). The diagram's own convention elsewhere is to annotate elided edges ("+ DB/mTLS — not drawn", and two such notes in `lfi-pipeline-v1`'s cards). Here they are silently absent, so the diagram reads as an under-specified network policy — exactly the artifact a security reviewer will use to write firewall rules.

### 1.5 — `ag_preexit` typed as "Deterministic tool / rule-based service" while carrying an LLM — **Low**
`lfi-pipeline-v1` and `-deployment-v1` both type Approval Coordinator `cloud` (legend: "Deterministic tool / rule-based service — **no LLM involved at all**") with the tag "+ guarded LLM assist". `lfi-ag-feedback-review-v1` types the same agent's summarize node `messagebus`. The legend's own wording makes `cloud` factually wrong here. Cosmetic, but it's the one box whose LLM usage is the subject of an entire ADR (0017).

### 1.6 — Context diagram draws notice ingestion as a system-to-system feed — **Low**
`lfi-pipeline-context-v1` has `notice_source → pipeline` "publishes notices/clauses" as an external-system relationship. ADR-0029 reversed this: notices are **uploaded by a human Lead/Approver in-app**. At C4-context zoom the actor is a person in the Settings/Admin screen, not a publishing system.

### 1.7 — Auditor/Compliance Reviewer absent from the context diagram's actor set — **Low**
ADR-0024 creates a third human role with standing read access to the most sensitive store in the system. `lfi-pipeline-context-v1` lists only `examiner_team` and `assistant_governor`. A context diagram whose job is "every external actor" should have it.

---

## Dimension 2 — Workflow / state-machine correctness

Scope: 3 LangGraph phase diagrams + 9 Examiner Dashboard workflow diagrams, against `agent-specifications.md`'s shared-state table and HITL table.

### 2.1 — ADR-0003's checkpoint (c) — "before anything is submitted for Assistant Governor approval" — exists in no checkpoint table and no workflow node — **Critical**
ADR-0003 is the system's load-bearing regulatory posture and names **exactly three** mandatory sign-offs, calling them "a deliberate, hard-to-reverse boundary… Removing a checkpoint later would be a regulatory-posture change, not a feature flag flip." `lfi-pipeline-v1`'s HITL card repeats all three ("Supervision-question list, findings/severity, and **pre-AG submission** require Lead/Approver sign-off").

The canonical HITL table in `agent-specifications.md` has **no pre-AG-submission row**. Its "AG review loop" row is resolved by the *Assistant Governor*, not a Lead/Approver, and is the AG's own decision — not a pre-submission gate. `lfi-workflow-phase3-postexam-v1` routes `findings_signoff → ag_showcase` directly.

This is not a wording quibble: `agent-specifications.md`'s Findings tab entry states "the sign-off checkpoint covers `findings[]` data, **not the rendered document**." So under the spec as written, **no human ever signs off on the transmittal letter or pre-exit deck before it reaches the Assistant Governor.** Either ADR-0003 is wrong and must be amended (a regulatory-posture change requiring an explicit decision), or the table and Phase-3 graph are missing a mandatory `interrupt()`. An engineer cannot resolve this from the artifacts.

### 2.2 — The AG redraft loop and the pre-exit revision loop — the entire subject of ADR-0010 — are drawn in no LangGraph diagram — **High**
ADR-0010 supersedes ADR-0002 specifically to make these "real state the pipeline tracks… not just a flat status field", and warns that building the flat version "would have meant reworking the state model almost immediately."
- `lfi-workflow-phase3-postexam-v1`: `ag_feedback_review` is a **dangling node with zero outgoing edges**; card says "redraft loop not drawn". `preexit_meeting` is terminal; card says "Pre-exit concerns → exit deck, loops back" — the loop exists only in prose.
- `lfi-ag-feedback-review-v1` does draw `redraft → awaiting_ag`, but it is a Dashboard-UX diagram, not the LangGraph graph, and its card asserts the pre-exit loop "reuses this exact flow" — which is not true at the state level (`preexit_status` has its own distinct 4-value vocabulary and its trigger is the LFI, not the AG).

Net: the two loops ADR-0010 says are *routine* and *the reason the state model exists* have no drawn state machine anywhere. ADR-0023 §2 mandates integration tests that "verify the AG revision loop actually routes back to Findings Author rather than silently dropping state" — there is no specified routing to test against.

### 2.3 — The AG/pre-exit redraft path bypasses the findings/severity sign-off checkpoint that ADR-0020 explicitly requires it to hit — **High**
`lfi-ag-feedback-review-v1` routes `feedback_review (confirmed) → redraft (Findings Author) → awaiting_ag`. There is no findings sign-off node in that path. ADR-0020 §3 is unambiguous: a redraft re-evaluates severity against the *current* rubric version, and "any redraft where `rubric_version` differs from the original draft's is called out explicitly to the Lead/Approver **at the findings/severity sign-off checkpoint**, not silently re-applied." `agent-specifications.md` repeats this verbatim under Findings Author's guardrails.
So a redraft can change a finding's severity and deadline — the two fields that become regulatory obligations on the bank — and go straight back to the AG without the human checkpoint the ADR names. This is the single highest-consequence workflow defect I found.

### 2.4 — The pre-meeting supervision-question drafting node and its mandatory `interrupt()` appear in neither Phase 1 nor Phase 2 — **High**
`agent-specifications.md` assigns pre-meeting drafting and its "**required sign-off** … LangGraph `interrupt()` here" to **Meeting Facilitator (a)**, and the HITL table's "Supervision-question sign-off" row says *Raised by: Meeting Facilitator*.
- `lfi-workflow-phase1-preexam-v1` puts "Draft Supervision Qs" in the **Compliance Analyst** lane and terminates at "Supervision Qs Ready → Phase 2".
- `lfi-workflow-phase2-meeting-v1` **begins** at "Phase 1 Complete — Qs finalized" and its first real node is the live meeting.
- `lfi-supervision-signoff-v1` puts the drafting lane under **Compliance Analyst** too.

The Meeting Facilitator's entire responsibility (a) — a node, an `interrupt()`, and an agent handoff — falls in the seam between two phase diagrams and is drawn in none of them, while three diagrams attribute its output to a different agent than the spec does.

### 2.5 — Phase 2's diagram collapses the Further-Submission Review checkpoint into an edge label — **High**
ADR-0029 introduces Further-submission review as a **new mandatory HITL checkpoint** ("a genuine gap… added to the HITL table, same per-item accept/edit/reject pattern as every other checkpoint"). In `lfi-workflow-phase2-meeting-v1` it is not a node: edge `requests-sufficiency` carries the label "reviewed, relayed, round-2 received", collapsing a mandatory `interrupt()`, a manual human relay, and a full second intake cycle into one arrow. The canonical phase-2 LangGraph diagram — the artifact `agent-specifications.md` §5 tells implementers to "read before implementing a phase" because prose "alone does not make [it] unambiguous" — omits a checkpoint. It exists in `lfi-further-submission-review-v1`, but that is a Dashboard flow, not the graph.

### 2.6 — `all_submitted` has two mutually exclusive definitions inside the same document — **Critical**
`agent-specifications.md` §Intake Tracker: "fires the `all_submitted` event only when **every RFI question's status is `submitted`**".
`agent-specifications.md` §Definition of done: "`all_submitted` fires only when **100% are non-`pending`**".
`suspicious` is non-`pending` but not `submitted`. Under reading A, one unparseable filename blocks gap analysis for the entire examination indefinitely; under reading B, gap analysis runs over documents that failed the format check. `lfi-intake-tab-v1` models only a "Marked Submitted" terminal and draws no `all_submitted` trigger at all, so it arbitrates nothing. An engineer literally cannot implement the trigger condition correctly.

### 2.7 — The insufficient-sufficiency loop-back is unimplementable as specified — **High**
`agent-specifications.md` §Meeting Facilitator (c): "`insufficient` re-enters Phase 1's Intake Tracker **via the same `all_submitted` trigger** for another document-collection round." But `all_submitted` is defined over `rfi_store`'s RFI question list (Intake Tracker's inputs are "`rfi_store` (expected question → folder/filename convention)"). Round-2 documents answer `further_submission_requests[]` items, which are **not** RFI questions and have no entry in `rfi_store`, no expected filename convention, and no `intake_status` vocabulary of their own. Nothing in the state table or any diagram says how a further-submission request becomes a trackable expected-document row. `lfi-further-submission-review-v1` asserts "Round-2 documents still flow through the same automated Intake Tracker (decision 100)" without closing this. Two engineers will build two incompatible things here.

### 2.8 — No reject/redraft edge on any of the three per-item sign-off diagrams — **Medium**
`lfi-supervision-signoff-v1`, `lfi-further-submission-review-v1`, and `lfi-findings-signoff-v1` all offer "accept / **edit** / **reject**" and then draw a single edge "every item disposed → finalized". Where does a *rejected* item go? For supervision questions, a rejection presumably drops the question; for a finding, rejecting it is materially different (does the finding disappear, or does Findings Author redraft?). The human-agreement metric (ADR-0016) depends on reject being a distinct, recorded outcome, yet no diagram gives it a destination state.

### 2.9 — Corpus-unavailable deferral state is defined in the tool contract but appears in no workflow — **Medium**
`tool-contracts.md` §`notice_corpus.get_clause`: on retry exhaustion "the clause-verdict loop iteration… is **deferred (re-queued, not failed-closed)** and surfaced on the Examiner Dashboard as a corpus-availability issue **distinct from a grounding gap**." `lfi-workflow-phase1-preexam-v1` has only two branches out of `gap_analysis` (grounded → verdict; ungrounded → Human Review Flags) and its card says "only a real absence… reaches Human Review Flags" — leaving the deferred/re-queued state with no node, no queue, and no Dashboard surface anywhere in `agent-specifications.md`'s Gap Analysis tab description (which lists exactly two resolution options, per `lfi-gap-analysis-resolution-v1`'s "Two Options, Not Three").

### 2.10 — `voided` examination state has no field, no transition, and no UI action — **High**
ADR-0024 §4 makes voiding a real, policy-bearing state: "a voided examination is flagged `voided` in its state and excluded from active Examiner Dashboard views/queues, but its `audit_log[]` entries remain queryable." There is no `voided` field in the shared-state table, no transition in any workflow, and no void action in `agent-specifications.md`'s Dashboard section. `lfi-examiner-dashboard-userflow-v1` models exactly one terminal (`auto_close` on `proceeded_to_exit`) and the portfolio filter is "Active/Completed/All" — with no place for voided. A regulator's data-retention policy that cannot be exercised is not a policy.

### 2.11 — `intake_status` "suspicious" has no resolution path other than override — **Medium**
`lfi-intake-tab-v1` branches `status_check → override` on "false negative" and `→ marked_submitted` on "ok". There is no modelled path for a genuinely malformed submission (a true positive) — no "request resubmission", no terminal `suspicious` state. Combined with 2.6, `suspicious` is a state the system can enter and has no documented way to leave except a Lead/Approver asserting it is fine.

---

## Dimension 3 — Sequence / interaction correctness

Scope: `lfi-guardrail-citation-grounding-v1`, `lfi-notice-corpus-manager-sequence-v1`, `lfi-setup-tab-sequence-v1`, cross-checked against `tool-contracts.md`.

### 3.1 — Two of the three sequence diagrams are drawn entirely against tools that have no contract — **High**
`lfi-notice-corpus-manager-sequence-v1` calls an "OCR + Segmenter" tool and writes confirmed clauses to Notice Corpus. `lfi-setup-tab-sequence-v1` calls an "RFI Parser" tool and writes to RFI Store. **None** of `ocr_segment`, `rfi_parse`, `notice_corpus.write/confirm`, or any RFI-store write appears in `tool-contracts.md`, whose stated purpose is that "every tool named in `agent-specifications.md` gets a real contract here." The corpus-manager gap is at least honestly flagged (`agent-specifications.md`: "Open item: scoped as a direction, no tool contract yet"; decision-log open items). **The RFI-parser gap is not flagged anywhere** — and it sits on the critical path of the very first screen an examiner uses. What does the parser return on a malformed Excel? On a question whose clause reference doesn't resolve in Notice Corpus? Nothing says.

### 3.2 — `log_checkpoint()` — the audit trail's only write path — has no contract and no record schema — **Critical**
`agent-specifications.md` §Audit Trail Store: every agent writes "via a shared `log_checkpoint(state_before, state_after, actor)` call". ADR-0018 §2 requires it to carry an idempotency key. ADR-0016 requires entries to be **hash-chained**. ADR-0016 requires accept/edit/reject **plus the edit diff** recorded per checkpoint. `agent-specifications.md` requires `insufficient_grounding` vs. low-confidence-supersession to be "logged as **distinct reasons** in `audit_log[]`", and `lfi-intake-tab-v1` requires overrides logged as "a **distinct event type**", and `lfi-setup-tab-sequence-v1` requires a distinct "setup corrected" event, and `lfi-gap-analysis-resolution-v1` requires human-asserted verdicts logged distinctly from model-asserted ones.

That is at least six different, load-bearing requirements on a record whose schema, event-type enumeration, hash-chain construction, and idempotency-key derivation are **specified nowhere** — not in `tool-contracts.md`, not in the shared-state table (`audit_log[]` has no field list), not in any ADR. This is the regulatory source of truth for the entire system (ADR-0007, ADR-0018: "silent double-processing… directly undermines the audit trail's claim to be the regulatory source of truth"). It is also the one artifact that is genuinely expensive to retrofit — a hash chain cannot be added to an existing log retroactively without invalidating it.

`lfi-guardrail-citation-grounding-v1` draws `gap_analysis → audit_trail` twice as a direct call, which is also a boundary inconsistency: the spec routes all audit writes through `log_checkpoint()`, not through direct DB messages.

### 3.3 — `ingestion_adapter.fetch_document()` is in ADR-0006's interface and used by the UI, but has no contract — **High**
ADR-0006 defines the adapter as "**list documents, fetch document, get last-modified**". `tool-contracts.md` specifies `list_documents` and `get_metadata` only. `agent-specifications.md`'s Intake tab requires "document access via **download/open through the Ingestion Adapter**", and `lfi-intake-tab-v1` draws an "Open Document" node. The fetch path — the one that streams potentially large, potentially hostile files from an LFI-controlled SFTP server to an examiner's browser — is the adapter's highest-risk operation and has no input/output/timeout/error contract, and no size limit.

### 3.4 — `lfi-setup-tab-sequence-v1` writes the SFTP root to RFI Store, while the spec puts it in the LFI registry — **Medium**
The sequence shows `examiner_app → rfi_store: "update SFTP root / start date"`. `agent-specifications.md` §Settings/Admin: "the **LFI registry** (name, license type, **SFTP root**, contacts; this app owns this data directly)". Two homes for the same field, with no statement of which wins when a per-examination override diverges from the registry value. Given ADR-0006's note that the SFTP structure is fixed per license type, the correct answer is probably registry-with-per-exam-override, but the artifacts don't say.

### 3.5 — Setup's initial creation writes no audit entry — **Medium**
`lfi-setup-tab-sequence-v1` logs only the *post-creation edit* ("setup corrected"). Examination creation — choosing the LFI, the start date, the SFTP root, and confirming the parsed RFI question list that everything downstream is measured against — writes to `rfi_store` with no `audit_trail` message. ADR-0007 requires the audit trail at every checkpoint and the Setup confirm is explicitly a human confirmation step ("setup finalizes only after this confirm").

### 3.6 — The guardrail sequence cites a "sequencing note" in `agent-specifications.md` that does not exist — **Medium**
`lfi-guardrail-citation-grounding-v1`'s retry card: "`supersession_graph.check()` runs after a grounded citation is confirmed, not before — **see `agent-specifications.md`'s sequencing note**." There is no such note; the only sequencing statement there is the ReAct shape "retrieve → verify citation → verdict", which doesn't mention supersession ordering at all. The ordering is load-bearing (checking supersession before grounding would waste calls; after, it can invalidate a just-confirmed citation) and currently exists only as an unsourced claim on a diagram card.

### 3.7 — Retry-count mismatch between the guardrail sequence and the contract — **Low**
The sequence shows attempt 1 → timeout → "attempt 2 (bounded)" → recovered. `tool-contracts.md` specifies **4 attempts** at 1s/2s/4s/8s for `get_clause`. The diagram is illustrative and says "full policy in tool-contracts.md", so it's not a contradiction — but combined with 2.9 (the exhaustion branch missing entirely), the diagram under-teaches the one behavior it exists to teach.

### 3.8 — No sequence diagram exists for the highest-risk interaction in the system — **Medium**
There are sequence diagrams for citation grounding, notice upload, and examination setup. There is none for `interrupt()` resolution — the path that carries ADR-0018's optimistic-concurrency etag, the idempotency key, the `log_checkpoint()` write, and the LangGraph resume. ADR-0023 defers "full TDD-style sequence diagrams for every internal REST call" (correctly), but this one is not a routine REST call; it is the mechanism the entire augmentation posture rests on.

---

## Dimension 4 — Cross-artifact consistency

### 4.1 — `severity` has two incompatible value sets — **Critical**
`CONTEXT.md` §Severity: "A finding's rated importance — **High / Medium / Low**". ADR-0020 and the rubric are built on it. `tool-contracts.md` §`severity_rubric.lookup` returns `"low"|"medium"|"high"|**"critical"**`. Four values vs. three. The rubric editor UI, the transmittal-letter template, the `findings[]` schema, and the severity guardrail's equality check all depend on this enum. `CONTEXT.md` is the repo's declared domain authority (CLAUDE.md routes domain language to it) and the tool contract is the build authority; they flatly contradict. Also note the casing differs, which matters for a code-enforced equality assertion (ADR-0017: "persisted `severity` must **equal** the last lookup's return value").

### 4.2 — "Clarification question" vs. "supervision question" — `CONTEXT.md` forbids the term five ADRs use — **Medium**
`CONTEXT.md` §Supervision question: "_Avoid_: **clarification question**, follow-up question". Yet ADR-0001 ("clarification-question sign-off"), ADR-0002 ("the pre-meeting clarification-question list"), ADR-0003 ("(a) the final clarification-question list"), ADR-0008 ("clarification-question review"), and ADR-0009 ("drafting clarification/follow-up questions") all use the forbidden term, as does `lfi-pipeline-deployment-v1`'s sublabel "Clarification & Meeting" and `lfi-workflow-phase2-meeting-v1`'s node "Live Exam Meeting — clarification Q&A". ADR-0003 is the checkpoint-defining ADR, so the term mismatch lands squarely on the artifact where checkpoint identity matters most (see 2.1).

### 4.3 — HITL checkpoint count is stated as 5 in two architecture diagrams and 3 in `CONTEXT.md`; the table has 8 — **Medium**
- `agent-specifications.md` HITL table: **8 rows**.
- `lfi-pipeline-deployment-v1` violet card: "**5 checkpoints** with SLAs — table in agent-specifications.md".
- `lfi-pipeline-context-v1` cyan card: "**5 HITL checkpoints** (agent-specifications.md)".
- `CONTEXT.md` §Human-in-the-loop checkpoint: names **3** plus revision loops.
- `lfi-pipeline-v1` violet card: names **3** plus loops.
Three different counts across five artifacts pointing at each other. The diagrams that cite "5" were not updated when ADR-0029 (further-submission review) and the sufficiency/AG-feedback checkpoints were added.

### 4.4 — "Two-role RBAC" is reaffirmed in ADR-0029 while the system has three roles — **Medium**
ADR-0024 §3 adds Auditor/Compliance Reviewer as "a new RBAC role… alongside Examiner and Lead/Approver". ADR-0029 then states "**Two-role RBAC (decision 35) was reconsidered and reaffirmed** during this round, not changed — worth recording here because the entire checkpoint/guardrail architecture above assumes it holds" — two paragraphs after describing the Auditor's distinct landing page. ADR-0007's title is still "**Two-tier** RBAC". An implementer reading ADR-0007 and ADR-0029 builds two roles; one reading ADR-0024 and `lfi-audit-log-scope-v1` builds three.

### 4.5 — Agent node IDs across every diagram still carry the pre-rename persona names — **Low**
`gap_analysis` = Compliance Analyst, `clarification_agent` = Meeting Facilitator, `findings_reporting` = Findings Author, `ag_preexit` = Approval Coordinator. Consistent across diagrams (good), but they will become the service names, module names, and OTel span names in code, and none of them matches the persona a reader learns from `agent-specifications.md`. Cheap to fix now; annoying forever after. Same class: "Lead-Approver" vs. "Lead/Approver" still splits across `lfi-pipeline-context-v1`, ADR-0015, ADR-0029 and `docs/decision-log.md`.

### 4.6 — Rubric scope stated three different ways — **Medium**
`agent-specifications.md` §Settings/Admin: "the severity/deadline rubric editor — **global, per license type**" (internally ambiguous on its own). `lfi-rubric-editor-v1` card title: "**One rubric per license type, not per-examination**". ADR-0020: a single shared versioned service with no scoping dimension mentioned. `tool-contracts.md` §`severity_rubric.lookup(finding, rubric_version?)` takes **no `license_type` parameter** — so as contracted, per-license-type rubrics cannot be looked up at all. (v1 is banks-only per ADR-0002, so this doesn't bite immediately — which is exactly why it will ship wrong.) See also 6.5.

### 4.7 — Rubric version history is promised from a screen that cannot show it — **Medium**
`lfi-rubric-editor-v1` and `agent-specifications.md`: "no browsable version-history UI — **full history stays queryable via Audit Log**". But `lfi-audit-log-scope-v1` and ADR-0029 scope the Audit Log tab as *per-examination* for Examiner/Lead-Approver, with portfolio-wide access reserved to the Auditor role. Rubric edits happen in Settings/Admin, **outside any examination**. So the Lead/Approver who edits the rubric has no route to its history, while the Auditor — who cannot edit it — does. The one role that needs it is the one role locked out.

### 4.8 — `ag_feedback_reviewed` state has no corresponding node in the Phase-3 graph's edge set — **Low**
`ag_status` is enumerated identically in `agent-specifications.md` and `lfi-ag-feedback-review-v1` (5 values, well done). But `lfi-workflow-phase3-postexam-v1` draws `ag_feedback_review` with no outgoing edge, so the transition into `ag_feedback_reviewed` — and out of it — is drawn in the Dashboard diagram only (see 2.2).

---

## Dimension 5 — Non-functional readiness

Summary of what is genuinely covered: security/isolation (ADR-0019), observability/eval (ADR-0016), testing/canary/rollback (ADR-0023), retention/lifecycle/audit access (ADR-0024), capacity methodology (ADR-0022), state-schema versioning (ADR-0025). This is a stronger non-functional story than most systems get at design review. The gaps below are what's left.

### 5.1 — No treatment of prompt injection via LFI-authored documents or meeting minutes — **High**
The pipeline feeds **adversary-authored content** — documents uploaded by the regulated bank under examination, which has a direct material interest in the verdict — into an LLM whose output drafts regulatory findings against that same bank. Meeting minutes are typed free-text; AG feedback is typed free-text. Nothing anywhere in 29 ADRs, `agent-specifications.md`, or `tool-contracts.md` mentions prompt injection, input sanitization, instruction/data separation, or output-boundary checks on untrusted content. ADR-0019 explicitly scopes out "malicious *file upload*" (a malware/parser concern) — which is not the same threat and doesn't cover it.
The mitigating factor is real and worth stating: ADR-0004's grounding rule means an injected instruction cannot fabricate a citation that doesn't exist in the corpus, and every output passes a human checkpoint. That materially bounds the blast radius — but it does not cover severity inflation/deflation, supervision questions being steered away from a real gap, or a "compliant" verdict on a clause that does exist. This deserves at minimum a named, explicit deferral with a revisit trigger, the way everything else in this repo gets one. It currently has neither.

### 5.2 — No language/localization treatment for a UAE regulator — **High**
Zero occurrences of Arabic, bilingual, localization, RTL, or language-handling across `CONTEXT.md`, all 29 ADRs, `agent-specifications.md`, `tool-contracts.md`, and `docs/decision-log.md`. For CBUAE this is not a polish item, it is an architecture input in at least four places: (a) OCR of scanned notices (ADR-0005) — Arabic OCR quality is materially different from English; (b) the citation-grounding rule requires **verbatim exact-match** of a clause sentence (ADR-0004) — verbatim matching across scripts, diacritics, and Unicode normalization forms is a real engineering problem, and the guardrail is an equality assertion; (c) model selection (decision 49, Qwen Coder vs. gpt-oss-120b) — Arabic legal-text competence is a first-order selection criterion that ADR-0013's "model-agnostic interface" framing sidesteps; (d) the transmittal letter and pre-exit deck templates. If the answer is "everything is English-only, confirmed with FPSD", that is a fine answer — but it is currently an unexamined assumption, not a decision.

### 5.3 — No backup/DR/RTO/RPO for the Fraud Database — **Medium**
The Fraud Database holds the append-only, hash-chained audit trail that ADR-0024 designates "the durable source of truth", plus every LangGraph checkpoint for examinations that are in-flight for 30+ days with multi-week revision loops. The only statement anywhere is one incidental clause in ADR-0014 ("CBUAE's existing Oracle DBA expertise, **backup/HA tooling**… already cover this new store"). That is an assumption about someone else's tooling, not a recovery objective. Losing a week of checkpoints means losing signed-off human decisions; there's no stated RPO for that.

### 5.4 — Testing story covers agents and graph, but not the application — **Medium**
ADR-0023 is strong on tool error paths, LangGraph resume, model canary, and in-flight pinning. It says nothing about the Examiner Dashboard: no end-to-end/UI test strategy, no RBAC authorization test requirement (the entire security model is "gated actions on one screen" — a class of bug that is invisible without explicit negative tests that an Examiner cannot resolve an `interrupt()` and an Auditor cannot write), and no application rollback story distinct from model rollback (ADR-0025 mandates forward migrations but names no down-migration or rollback path for a bad deploy against in-flight checkpoints).

### 5.5 — SLAs are "suggested" with no configuration or enforcement mechanism — **Medium**
The HITL table's SLAs drive Dashboard breach flags and the portfolio landing page, but are labelled "Suggested SLA", with one row "Not yet set", one "hard deadline, not a duration" (i.e. a different data type — it's the meeting date, which lives in no state field), and one "no pipeline-enforced SLA". Are these constants, config, or per-examination data? Where does the meeting date come from — Setup doesn't capture it. Two engineers build this two ways.

### 5.6 — No document-content storage decision, flagged only inside a capacity placeholder — **Medium**
ADR-0022 notes in passing that whether the Fraud Database "stores document content or just pointers into the Shared Workspace/Smart Portal… is **not yet decided**". That is not a capacity detail — it's a data-architecture decision with retention (ADR-0024), encryption (ADR-0019), and evidentiary consequences: if the pipeline only holds pointers, and the Shared Workspace is *being delisted* (`CONTEXT.md`), then the evidence underlying a signed-off regulatory finding can disappear from under the audit trail. It deserves its own decision, not a parenthetical.

---

## Dimension 6 — Build-readiness gaps

### 6.1 — The canonical state schema cannot support the Examiner Dashboard as specified — **Critical**
`agent-specifications.md`'s shared-state table is declared the canonical state contract and is scoped "per examination (one instance per LFI per examination cycle)". The Dashboard, described in detail in the same document and in ADR-0029, reads fields that **do not exist in it**:

| Dashboard feature (source) | Field needed | In state table? |
|---|---|---|
| Portfolio home: LFI, license type (ADR-0029) | `lfi_id`, `license_type` | **No** |
| Portfolio home: current phase | `phase` / `current_phase` | **No** |
| Portfolio home: SLA countdown + breach flag | checkpoint due timestamps | **No** |
| Portfolio home: open-checkpoint count, last activity | checkpoint registry, timestamps | **No** |
| Active/Completed filter, auto-close | `examination_status` | **No** |
| "reach it only within an examination they're **assigned to**" | assignment/membership | **No** |
| Voided examinations (ADR-0024) | `voided` | **No** (see 2.10) |
| Setup: start date, SFTP root | — | **No** (see 3.4) |
| Model pinning for in-flight exams (ADR-0023 §4) | `model_version` | **No** |
| RFI question list itself | `rfi_questions[]` | **No** (only `rfi_responses[]`) |

The table has 14 rows and is missing the identity, lifecycle, ownership, and scheduling fields the landing page is built on. An engineer starting on the Dashboard today must invent an entire examination-record schema. This is the single largest build-readiness gap.

### 6.2 — `audit_log[]` has no field-level schema — **Critical**
See 3.2. Called out separately here because it is a build artifact, not just a sequence-diagram omission: `audit_log[]` appears in the state table as a bare name, with six distinct downstream requirements (hash chain, idempotency key, actor, distinct event types, accept/edit/reject + diff, distinct reason codes) and no definition of a single field.

### 6.3 — `findings[]`, `compliance_verdicts[]`, and `supervision_questions[]` have no element schemas — **High**
The state table names them as arrays; only `findings[]` gets a partial parenthetical ("severity, deadline, `rubric_version`, citations"). But `tool-contracts.md` references `FindingSummary`, `FindingRecord`, and `EdmQueryRef` as **named types that are defined nowhere**, and `template_fill`'s error `schema_mismatch` is defined as "a `FindingRecord` is missing a field the fixed Word/PPT template requires" — a validation against a schema that does not exist in the repo. The citation sub-structure that ADR-0004's whole guardrail checks against (clause_id + verbatim text + notice ref + which sentence) is similarly unspecified. A code-enforced equality guardrail needs a typed record to enforce against.

### 6.4 — In-app v1-scope screens with no implementable substrate — **High**
Three screens are in v1 scope per ADR-0029 and drawn in diagrams, with no tool contract and no state entry: **Notice Corpus Manager** (flagged as an open item — honest, but it is a hard prerequisite for ADR-0004's grounding rule, so it is on the critical path, not the backlog), **RFI Parser / Setup** (not flagged anywhere — see 3.1), and the **LFI registry** (named once in one sentence of `agent-specifications.md`; no schema, no diagram, no contract, and it owns the SFTP root every ingestion depends on).

### 6.5 — `severity_rubric.lookup()`'s signature cannot express the rubric's own scoping — **Medium**
See 4.6. `lookup(finding, rubric_version?)` with `finding: FindingSummary` (undefined type, 6.3) and no `license_type`. Per-license-type rubrics are unreachable through the contracted interface. Adding a parameter later is cheap; discovering it after the rubric-version guardrail is code-enforced across `findings[]` is not.

### 6.6 — The "one physical Fraud Database" decision has no schema artifact — **Medium**
ADR-0012 puts notice corpus, audit trail, RFI store, and analysis output in one Oracle instance with logical separation that stays "load-bearing" (per-clause queryability, append-only guarantee). There is no ER diagram, no table list, no statement of how append-only is enforced in Oracle (trigger? grant model? insert-only role?). The decision log already names an ER diagram as a known-missing next artifact — correctly — but ADR-0012's append-only *guarantee* is an implementation claim that currently has no implementation.

### 6.7 — No API contract between the Examiner Dashboard and the backend — **Medium**
The deployment diagram draws one edge labelled "internal REST API". Every HITL checkpoint resolution, every per-item accept/edit/reject, the etag/optimistic-concurrency protocol (ADR-0018 §3), and the idempotency-key propagation all cross this boundary. ADR-0023 defers "full TDD-style sequence diagrams for every internal REST call" — reasonable — but the *checkpoint-resolution* endpoint's concurrency semantics are a correctness requirement an ADR already made, not a routine CRUD call. Frontend and backend will be built by different teams (per the deployment diagram's own `tag` fields) against no shared contract.

---

## Correctly deferred — not a gap

These are explicitly scoped-out, with a reason and (mostly) a revisit trigger. I am counting them **in the project's favour**; the deferral discipline here is better than most teams manage.

- **Content/plausibility validation agent** — ADR-0002, decisions 18/28.
- **Unified cross-feature review queue** — decision 31.
- **Effective-dated notice versioning** — ADR-0005, decision 32, with a *resolution path* pre-specified in ADR-0027 (relational `effective_from`/`effective_to`, not graph/RAG). Exemplary.
- **SVF and non-bank license types** — decision 17.
- **Dedicated Verifier agent** — ADR-0026, with the DS-STAR analysis and a narrower draft-completeness pre-check left open against a concrete observed signal.
- **Adverse-media / consumer-complaint monitoring** — ADR-0028, blocked on a named external legal answer.
- **Sanctions-screening tool** — ADR-0028, scoped as direction, contract acknowledged missing.
- **Capacity numbers** — ADR-0022, methodology given, placeholders explicitly marked, named next step (ask FPSD for concurrency).
- **Supersession confidence threshold; HITL SLA durations; further-submission SLA** — "tune empirically", decisions 30/54/99.
- **Deployment platform (VMs vs. K8s), secrets-manager product, IdP choice, BI tool** — all four given the same consistent "ask CBUAE's platform team, don't guess a product into the diagram" treatment.
- **Model choice (Qwen Coder vs. gpt-oss-120b)** — ADR-0013, with the swappability consequence carried through into ADR-0023's canary and pinning policy.
- **In-app metrics dashboard; in-browser letter/deck rendering; email notifications; local user management** — ADR-0029, each with a reason.
- **Full STRIDE threat model, ingestion-path malicious-file analysis, CBUAE security sign-off** — ADR-0019, deferred to a scheduled pre-go-live exercise. (Note: this deferral does **not** cover prompt injection — see 5.1.)
- **On-call runbook; per-REST-call sequence diagrams** — ADR-0023, correctly sequenced to before-go-live rather than before-implementation.
- **Langfuse retention period; audit-trail retention period; access-review cadence** — ADR-0024, pending data-governance/legal input, with a safe interim default stated.
- **Fraud Database ER diagram and a lightweight PRD** — already named in the decision log's open items as the next two SDLC artifacts. (I nonetheless rate the *state-schema* half of this Critical at 6.1 — the ER diagram is a nice-to-have; the examination record is a blocker.)
- **Cost model; independent model-risk sign-off** — organizational, named.

---

## Cross-check against the two prior reviews

Read only after the above was drafted. No finding above was altered as a result; two were strengthened by corroboration, and I record one material disagreement.

### `.scratch/architecture-review-2026-09-15.md` (whole-system, 17 findings)

Overlap in *kind*, almost none in *instance*, because its findings were all genuinely closed — ADR-0017 through ADR-0025 exist because of it, and I independently verified each closure holds.

**Where a closed finding left a residue I found open:**
- Its **High #6** ("tool contracts are prose, not contracts") produced `tool-contracts.md` — but only for the eight tools that existed that day. `log_checkpoint`, `ingestion_adapter.fetch_document`, the RFI parser and the OCR/segmenter were never brought under it (my 3.1/3.2/3.3/6.4). The finding was closed as an instance, not adopted as a rule.
- Its **Medium #12** produced ADR-0025's `schema_version` — versioning the state shape without anyone checking whether the shape is *complete* (my 6.1).
- Its **priority list item #3** explicitly named "a data model / ER diagram… `audit_log` structure **including the hash-chain fields from ADR-0016** — needed before DB implementation starts." That is my 6.2/6.6, raised two days ago and still open, now demoted in the decision log to a general "ER diagram" open item that loses the hash-chain specificity. **We agree, and I am escalating it to Critical** — it was under-rated as a "next SDLC artifact" when it is a blocker.

**One material disagreement, and it is a real one.** That review's per-agent tool analysis states Gap Analysis is "missing a defined sequencing contract that supersession check happens **before** a citation is finalized." `lfi-guardrail-citation-grounding-v1`'s card asserts the **opposite** order — "`supersession_graph.check()` runs **after** a grounded citation is confirmed, not before" — and cites a "sequencing note" in `agent-specifications.md` that does not exist (my 3.6). So the two artifacts now disagree on the ordering, no ADR adjudicated it, and the diagram claims an authority for its answer that isn't there. My 3.6 stands and this raises it: it is a genuine open technical question (checking supersession after grounding means a confirmed citation can be invalidated by a superseding clause found afterwards), not just a dangling cross-reference.

**Found here, not there** (mostly because it predates the Dashboard round): all of dimension 2, 4.1 severity enum, 5.1 prompt injection, 5.2 language, 2.1 the missing third ADR-0003 checkpoint.

### `.scratch/examiner-dashboard-coverage-review-2026-09-17.md` (Dashboard coverage, 7 findings, today)

Its findings were closed in commits `60e5092`/`580d60e`, which is why the 9 Dashboard diagrams are internally tidy. Verified: its #1 (`redraft` state) closed by a card note; #3 (parallel-UI fork shape) closed by collapsing to a single `Gated Actions` node; #4 (Setup post-creation edit) closed with a new segment; #5 (missing supervision sign-off diagram) closed by creating `lfi-supervision-signoff-v1`; #7 partially closed via the "unlocks by data" tag. Its #6 (Lead/Approver vs. Lead-Approver) is still live in `lfi-pipeline-context-v1`, ADR-0015, ADR-0029 and `decision-log.md` — folded into my 4.5 as the same class of issue.

**Material disagreement on its Finding 2.** It found `lfi-workflow-phase2-meeting-v1` contradicting decision 99's checkpoint and offered two fixes: insert the checkpoint node, **or** add a cross-referencing card note. The team took the card note. I think that was the wrong choice and I am re-raising it at High (my 2.5): a card note fixes a *reader's* confusion, not the artifact. The canonical Phase-2 LangGraph diagram — the one `agent-specifications.md` §5 instructs implementers to read *because prose alone is not unambiguous* — still has no node for a mandatory `interrupt()`, and the edge that replaces it now carries a three-step label that reads as a single transition. The prior review's own diagnosis ("an engineer building the actual LangGraph graph from this diagram would wire `extract` directly into the sufficiency node") is still true after the fix. Same objection applies to its Finding 5's closure: the diagram was created, but under the wrong agent lane (my 2.4).

**Where we diverge in method, and why it matters:** that review checked *diagram-to-ADR coverage* — is every decision drawn. I checked *diagram-to-implementable-substrate* — can the drawn thing be built. A coverage check cannot surface 6.1 (the state table cannot support the portfolio home it correctly validated as "covered"), 2.3 (the redraft path it validated bypasses a checkpoint ADR-0020 mandates), or 2.1 (a checkpoint absent from the canonical table — there was nothing to have coverage *of*). Its own scope note explicitly excludes re-litigating the 2026-09-15 findings, which is exactly where the residue in 6.1/6.2 was sitting.

### The pattern across all three reviews

This team closes findings *precisely as scoped* and has not once generalized one into a standing invariant. Three independent reviews have now each found a different instance of the same failure: **a diagram or ADR running ahead of the build-level artifact underneath it.** That is the thing to fix, not the eleven instances.

---

## Approval verdict

**Yes, with conditions. Development starts Monday on Phase 1 and the Setup/Intake screens; it does not start on Phase 3 or the Findings/Approval screens until six items close.**

I am not sending this back for another design round. The architecture is sound, the decision discipline is genuinely excellent, and the remaining defects are concentrated, nameable, and mostly closable in under a week by the people who already hold the context. Blocking the whole thing would waste that.

**Blocking conditions — must close before the code they govern is written:**

1. **Resolve ADR-0003's third checkpoint (2.1).** Either add a pre-AG-submission sign-off row to the HITL table and a node to the Phase-3 graph, or amend ADR-0003 with an explicit decision that findings sign-off subsumes it. This is a regulatory-posture question, and I want the amendment on the record either way. *Blocks Phase 3.*
2. **Draw both revision loops as real state, and route the redraft through findings sign-off (2.2, 2.3).** ADR-0020 already dictates the answer; the graph just has to say it. As drawn, a severity can change and reach the Assistant Governor with no human checkpoint. *Blocks Phase 3 and the Approval tab.*
3. **Specify the `audit_log[]` record and `log_checkpoint()`'s contract (3.2, 6.2).** Fields, event-type enum, hash-chain construction, idempotency-key derivation. A hash chain cannot be retrofitted. *Blocks everything — this is the first thing to write.*
4. **Extend the shared-state table to cover the examination record (6.1).** Identity, license type, phase, status, assignment, `voided`, model pin, start date, SLA timestamps, `rfi_questions[]`. *Blocks all Dashboard work.*
5. **Pick one `severity` enum (4.1)** and fix `CONTEXT.md` or `tool-contracts.md`. Ten minutes; it is code-enforced by an equality assertion, so it will fail loudly and late otherwise.
6. **Settle `all_submitted` (2.6) and the round-2 intake mechanism (2.7).** Both live in the first module anyone will write. *Blocks Phase 1.*

**Non-blocking, but on the board with owners before the next review:**

7. **A written position on prompt injection (5.1)** — even "accepted risk, bounded by ADR-0004 + HITL, revisit at security sign-off" is fine. It must not be the one risk in this repo with no ADR.
8. **A written position on Arabic / language handling (5.2)** — with OCR, verbatim-match normalization, and model selection called out as the three affected decisions.
9. **Tool contracts for RFI parser, OCR/segmenter, and `ingestion_adapter.fetch_document` (3.1, 3.3)**, plus the `FindingRecord`/`FindingSummary`/`EdmQueryRef` type definitions (6.3).
10. **Document content: stored or pointed-to (5.6)** — promote out of ADR-0022's parenthetical into its own decision, given the Shared Workspace is being delisted.
11. **Adjudicate the supersession-check ordering (3.6)** — the guardrail diagram and the 2026-09-15 review state opposite orderings and no ADR settles it. Write the sequencing note the diagram already claims exists.
12. **Consistency sweep (4.2–4.7)**: checkpoint counts, clarification/supervision terminology, two-vs-three roles, rubric scoping + `license_type` parameter, rubric-history access route.

**One process condition, and I mean this as the most valuable thing in this review:** three independent reviews have now each found a different instance of the same failure mode — a diagram or an ADR running ahead of the build-level artifact underneath it. Adopt it as a standing invariant, in the ADR-0017 style: *no diagram ships showing a tool, state field, or checkpoint that does not have a corresponding row in `tool-contracts.md` or the shared-state table.* Enforce it in review. That converts a recurring class of finding into a closed one, which is the thing this team has not yet done to itself.
