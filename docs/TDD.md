# Technical Design Doc: LFI Examination Pipeline

| | |
|---|---|
| **Version** | 1.1.0 |
| **Status** | Design complete, pre-implementation |
| **Last updated** | 2026-09-19 |
| **Companion doc** | `docs/PRD.md` (what and why, for a product/business reader) |
| **Reviewed by** | 4 independent architecture reviews (2026-09-15, 2026-09-17 x2, 2026-09-18) — `docs/reviews/` |
| **Approved by** | Not yet — no FPSD leadership/document-owner sign-off has been recorded against this design |

## Changelog

| Version | Date | Change |
|---|---|---|
| 1.0.0 | 2026-09-19 | Initial version, consolidating decisions 1–117 / ADR-0001–0032 |
| 1.1.0 | 2026-09-19 | Rewrote §8 Rollout to separate build order from a real launch plan (shadow mode, pilot, go/no-go gate, rollback trigger distinct from ADR-0023's model-swap canary); added an unfilled "Approved by" field. Per independent audit, `.scratch/prd-tdd-audit-2026-09-19.md` (not committed). |

This is the engineering-facing counterpart to `docs/PRD.md`. It describes the system as currently designed: components, data flow, contracts, and the cross-cutting concerns a reviewer would expect in a Google-style design doc. No code exists yet — this describes the target, distilled from `docs/decision-log.md` (117 decisions) and `docs/adr/0001`–`0032`. Treat those two as the source of truth if anything here goes stale; this document summarizes, it doesn't supersede.

---

## 1. Context and scope

FPSD's LFI examination process (see `docs/PRD.md` §2) is currently manual across three phases. This system automates document-tracking and clause-level compliance drafting while keeping every regulatory judgment behind a human sign-off (ADR-0003). v1 scope is bank-only LFIs, English-language content, on-prem deployment, all three phases end-to-end (ADR-0002).

## 2. Goals / non-goals

See `docs/PRD.md` §3–4 for the product framing. The technical corollaries that shape this design specifically:

- **No autonomous state transition may skip a human checkpoint** — this is enforced in code (guardrail pattern, ADR-0017), not by convention.
- **No compliance verdict without a grounded citation** — enforced as a hard rule in the Compliance Analyst agent and the `notice_corpus` tool contract (ADR-0004), the single most load-bearing constraint in the system.
- **Model-agnostic LLM interface** — v1 does not commit to a specific self-hosted model (ADR-0013, decision 49); swapping models must not require touching agent logic.
- **No new user/identity store** — roles are claims on CBUAE's federated SSO identity (decision 46).

## 3. System overview

### 3.1 Orchestration

**LangGraph** (ADR-0001) — chosen for explicit node/edge/conditional-transition control and native `interrupt()` support, which maps directly onto the HITL checkpoint model. One shared graph state object per examination, carrying a `schema_version` field; any breaking state-shape change requires a tested migration function before shipping (ADR-0025). Full state schema, including `findings[]`, `compliance_verdicts[]`, and `supervision_questions[]` element shapes, is defined in `docs/architecture/agent-specifications.md` §"Shared graph state".

The graph is phase-segmented into three logical workflows (Phase 1 pre-exam, Phase 2 meeting, Phase 3 post-exam) — not one monolithic graph — because the diagramming tool's workflow grammar caps at 6 logical columns per lane, which happens to match a real, useful decomposition boundary (decision 55). See:
- `docs/architecture/lfi-workflow-phase1-preexam-v1.html`
- `docs/architecture/lfi-workflow-phase2-meeting-v1.html`
- `docs/architecture/lfi-workflow-phase3-postexam-v1.html`

### 3.2 Agents and services

**Correct framing (decision 73, don't say "5 agents"): 4 LLM agents + 1 rule-based service**, plus supporting platform services.

| Component | Type | Role |
|---|---|---|
| **Intake Tracker** | Rule-based service, not an LLM | Tracks document submission status per RFI question against the fixed per-license-type folder structure |
| **Compliance Analyst** | LLM agent + tool calls | Produces compliance verdicts, supervision-question drafts, strength/weakness summary — citation-grounded (ADR-0004) |
| **Meeting Facilitator** | LLM agent + tool calls | Drafts pre-meeting supervision questions; post-meeting, extracts `further_submission_requests[]` from minutes text |
| **Findings Author** | LLM agent + tool calls | Drafts findings (qualitative + quantitative tracks), applies the severity rubric, fills transmittal letter / pre-exit deck templates |
| **Approval Coordinator** | LLM agent + tool calls | Tracks AG status, summarizes raw AG feedback into redraft notes (shown side-by-side with the raw text, never silently applied) |
| **Notice Corpus Manager** | Cross-cutting service | OCR + clause segmentation on ingested notices, human-reviewed before anything becomes citable |
| **Audit Trail Store** | Cross-cutting service | Append-only, hash-chained log of every agent-proposed vs. human-approved/overridden state at every checkpoint |
| **Examiner Dashboard** | UI | Single unified app (ADR-0008), RBAC-gated actions on shared screens (decision 89) |

Full per-agent contracts (trigger, inputs, outputs, tools, hard rules, HITL checkpoints with SLAs, failure/retry policy, observability metrics, definition-of-done) are in `docs/architecture/agent-specifications.md` — read that before implementing any agent. Agent display names (Compliance Analyst, Meeting Facilitator, Findings Author, Approval Coordinator) are reader-facing personas; internal/functional names are preserved in diagram sublabels (decision 76).

### 3.3 Data architecture

**Fraud Database** — one physical Oracle store (ADR-0014), four logical sub-stores (ADR-0012, extended decision 71):
1. Notice Corpus (ingested, OCR'd, clause-segmented regulatory text)
2. RFI Store (parsed RFI question banks, per examination)
3. Audit Trail Store (hash-chained checkpoint log)
4. AI-generated analysis output (verdicts, supervision questions, findings)

**Stores full document content, not pointers**, for anything Intake Tracker marks `submitted` — decided specifically because the source Shared Workspace is being delisted; a pointer into a system scheduled for delisting would let regulatory evidence disappear out from under its own audit trail (ADR-0032). Encrypted at rest (ADR-0019).

**EDM (Oracle, external)** — quarterly LFI fraud submissions. Read-only input to gap analysis and quantitative findings; never reconciled against document claims (ADR-0009, ADR-0011).

**Ingestion Adapter** (ADR-0006) — pluggable `list_documents` / `get_metadata` / `fetch_document` interface isolating downstream logic from the concrete transport. v1 concrete implementation is SFTP against the Shared Workspace; a Smart Portal (API or SFTP) adapter is a designed-for future swap requiring no downstream changes. Transport choice is platform-wide config, not a per-examination field (decision 87) — surfacing it per-examination would leak this abstraction into the UI.

### 3.4 Notice corpus and citation grounding

This is the design's central guardrail, so it gets its own subsection.

- **Hard rule (ADR-0004)**: a compliance verdict requires a verbatim, exact-match citation to a real notice clause. No confident grounded match → output is `insufficient_grounding, needs human review`, never a guess.
- **Storage strategy (ADR-0027, confirmed via external research in decision 79)**: a hybrid of `notice_corpus.search()` (discovery), `notice_corpus.get_clause()` (exact-match grounding), and `supersession_graph.check()` (single-hop relationship lookup) — not a pure RAG system. Rationale: legal-citation RAG hallucinates 17–33% of the time in published research, and a hallucination with a citation attached is worse than one without, since it looks trustworthy. `supersession_graph` is architecturally a lookup table wearing a graph-shaped API — sufficient because nothing in v1 needs multi-hop traversal.
- **No dedicated Verifier agent** (ADR-0026). DS-STAR-style (arXiv 2509.21825) Planner/Verifier patterns don't port: their Verifier judges a runnable, checkable artifact; this system's outputs are human regulatory judgments by design. The existing code guardrails plus HITL checkpoints already fill that role, better-fitted to a domain where the real arbiter must stay human.
- **Notice versioning**: v1 is current-version-only (decision 32) — a known, explicit gap. Resolution path when picked up: structured `effective_from`/`effective_to` relational columns, not a graph or RAG extension (decision 79).
- See the full retry-vs-real-absence call flow in `docs/architecture/lfi-guardrail-citation-grounding-v1.html`.

### 3.5 Tool contracts

Every tool the agents call has a real schema (input/output, distinct error shape — "tool unavailable" is never collapsed into "no result found"), a timeout, and a retry policy. Full contracts in `docs/architecture/tool-contracts.md`:

`log_checkpoint`, `ingestion_adapter.{list_documents, get_metadata, fetch_document}`, `rfi_parser.parse`, `ocr_segment`, `notice_corpus.{search, get_clause}`, `edm.query`, `supersession_graph.check`, `severity_rubric.lookup`, `quant_analysis.compute`, `template_fill`.

Two tools are deliberately deterministic, not LLM-computed, because getting them wrong has direct regulatory consequences:
- **`severity_rubric.lookup`** — a shared, versioned platform service (ADR-0020). Findings Author's persisted `severity` must equal what this returns, checked in code, not just prompted for. A redraft re-evaluates against the *current* rubric version; drift is surfaced explicitly, never silently applied.
- **`quant_analysis.compute`** — arithmetic on EDM figures (ratios, trends, volumes) is code, never an LLM computing directly from raw figures in-context (ADR-0021).

### 3.6 UI architecture

Single unified app (ADR-0008), one screen per examination tab, RBAC as gated *actions* on identical data — not parallel UIs per role (decision 89). Sidebar tabs: **Setup → Intake → Gap Analysis → Meeting → Findings → Approval → Audit Log**, unlocking by data availability rather than a forward-only phase gate, since AG/pre-exit revision loops aren't strictly linear (decisions 83, 88, ADR-0010). Portfolio home is the pre-examination landing page (decision 81). Full screen-by-screen behavior is in `docs/architecture/agent-specifications.md` §"Examiner Dashboard" and the userflow/sequence diagrams under `docs/architecture/` (see the gallery at `docs/index.html`).

## 4. Diagram index

| Diagram | File |
|---|---|
| C4 Context | `lfi-pipeline-context-v1.html` |
| Logical architecture (container-level) | `lfi-pipeline-v1.html` |
| Deployment & ownership | `lfi-pipeline-deployment-v1.html` |
| Phase 1/2/3 LangGraph workflows | `lfi-workflow-phase{1-preexam,2-meeting,3-postexam}-v1.html` |
| Examiner Dashboard userflow | `lfi-examiner-dashboard-userflow-v1.html` |
| Per-tab lane views | `lfi-{setup-tab,intake-tab,gap-analysis-resolution,supervision-signoff,further-submission-review,findings-signoff,ag-feedback-review,audit-log-scope,rubric-editor}-v1.html` |
| Per-tab sequence diagrams | same subjects, `-sequence-v1.html` suffix |
| Citation-grounding guardrail sequence | `lfi-guardrail-citation-grounding-v1.html` |
| Notice Corpus Manager sequence | `lfi-notice-corpus-manager-sequence-v1.html` |

## 5. Alternatives considered and rejected

| Alternative | Rejected because |
|---|---|
| Pure RAG over the notice corpus | Published legal-citation RAG hallucination rates (17–33%) unacceptable for a system whose entire value proposition is grounded citation; hybrid exact-match approach chosen instead (ADR-0027) |
| Dedicated LLM Verifier agent (DS-STAR-style) | Verifies runnable artifacts, not human regulatory judgments; would create an autonomous-approval path that conflicts with ADR-0003 (ADR-0026) |
| Pointer-only document storage in the Fraud Database | Source Shared Workspace is being delisted — pointers would let audit evidence disappear from under its own record (ADR-0032) |
| Third-party/cloud LLM API | Data sensitivity + on-prem deployment constraint rules this out; self-hosted, in-tenant only (ADR-0013) |
| Locked, forward-only phase-stepper navigation | Real process has routine AG/pre-exit revision loops that aren't linear; tabs unlocking by data availability fit better (decision 83, ADR-0010) |
| Per-examination transport picker (Shared Workspace vs. Smart Portal) | Would leak the Ingestion Adapter's abstraction into the UI; kept as platform-wide config (decision 87) |
| One combined LangGraph diagram for all three phases | Diagramming tool's workflow grammar caps at 6 logical columns per lane; also maps naturally onto the existing 3-phase process boundary (decision 55) |
| Single RBAC role (drop Examiner/Lead-Approver split) | Explicitly reconsidered and reaffirmed — the entire guardrail/HITL architecture (ADR-0003, 0017, 0020) assumes drafter ≠ approver; team-staffing realities don't require collapsing the concept (decision 109) |
| Unified cross-feature human-review queue | Deferred; per-feature review surfaces judged acceptable for v1 scale (decision 31) |

## 6. Cross-cutting concerns

### 6.1 Security

- Threat model + network isolation for the LLM Inference Service: dedicated security-group boundary, mTLS-only access from the 4 LLM agents, `secrets_manager` component, encryption at rest on the Fraud Database (ADR-0019).
- **Prompt injection** via LFI-authored content (documents, meeting minutes, AG feedback) is an accepted, bounded risk for v1, not an unexamined gap: citation-grounding limits the worst case to `insufficient_grounding`, and every LLM output passes a HITL gate before it's binding. Explicitly does **not** cover subtle severity-steering within otherwise-plausible language — flagged, with a revisit trigger tied to the pre-go-live security exercise and to human-agreement-rate anomalies (ADR-0030).
- Identity federates to CBUAE's existing SSO (OIDC/SAML) — no new user/credential store (decision 46).

### 6.2 Reliability: failure, retry, idempotency (ADR-0018)

- Tool-call retry with backoff — **a tool timeout/unavailability must never be treated as "no matching data"** under the citation-grounding rule (the single most important failure-semantics distinction in the system).
- Idempotency keys on state-mutating calls.
- Optimistic concurrency on `interrupt()` resolution (checkpoint sign-off race conditions).
- Schema-validation-with-repair on structured LLM output.
- A visible "sync degraded" banner surfaces only after 3 consecutive missed polling intervals — not on every transient retry, which backoff already absorbs silently (decision 117).

### 6.3 Observability and evaluation (ADR-0016)

- **Infra-level**: OpenTelemetry + Prometheus/Grafana/Loki/Tempo ("LGTM" stack). RED method per service, USE method for the GPU-backed LLM Inference Service.
- **LLM-specific**: self-hosted Langfuse — every agent's prompt/completion/tokens/cost linked to its OTel trace; hosts both automated and human-annotated evals on the same trace.
- **Primary trust metric: human-agreement rate** (how often a Lead/Approver accepts a draft unedited), not raw accuracy — a rate near 100% is an automation-bias flag, not a win.
- Audit consumption is query-based against the Fraud Database (`audit_log`, hash-chained for tamper-evidence) — deliberately no second reporting surface; a unified in-app metrics dashboard is an explicit v2 candidate, not built now (decision 116).

### 6.4 Data retention and lifecycle (ADR-0024)

- Langfuse trace retention is bounded and shorter than the Fraud Database audit trail (exact period is an open item for CBUAE governance).
- Voided examinations are flagged, never physically deleted.
- A third, read-only **Auditor/Compliance Reviewer** RBAC role exists specifically for retention/lifecycle-scoped access, portfolio-wide (decision 106).

### 6.5 Testing, canary, rollback (ADR-0023)

- Unit tests for tool error paths (distinguishing timeout/unavailable from genuine no-match).
- A LangGraph integration test suite covering interrupt/resume and guardrail-enforcement paths.
- A canary process for model swaps.
- **In-flight examinations are pinned to the model version they started on** — a model swap never silently changes the grounding behavior underneath an examination already in progress.

### 6.6 Capacity (ADR-0022 — explicit placeholder, not sized numbers)

GPU/storage figures are illustrative only, pending real annual examination volume and concurrency data from the examiner team. The pointers-only capacity estimate this ADR originally carried is **invalidated** by ADR-0032 (full document content storage) and needs redoing with real per-document-size assumptions once usage data exists. Do not treat any number here as a procurement input.

### 6.7 Internationalization

v1 targets English-language notices, RFI documents, and meeting minutes only — flagged, not assumed (ADR-0031). This bites in four concrete places if it changes: Arabic OCR quality/tooling differs materially from English; the citation-grounding exact-match check needs an explicit Unicode normalization step across scripts; model selection (Qwen Coder vs. gpt-oss-120b) becomes a first-order criterion rather than a swappable detail; and the fixed Word/PPT templates are implicitly English-layout. **Revisit trigger**: get explicit FPSD confirmation before Notice Corpus ingestion begins in earnest.

## 7. Deployment (ADR-0013/0014/0015, ADR-0019)

- **On-prem**, explicit and revisitable choice, not "cloud-agnostic, undecided" (decision 48).
- **Self-hosted, open-weight LLM, in-tenant** — no third-party model API call. Agents built against a model-agnostic interface; two named candidates (**Qwen Coder**, **gpt-oss-120b**), neither committed (decision 49).
- **Oracle** for the Fraud Database, matching EDM's existing engine — chosen for CBUAE's existing operational Oracle expertise over pure workload fit (decision 47).
- Container-orchestration boundary (VMs vs. K8s/OpenShift) still open.
- See `docs/architecture/lfi-pipeline-deployment-v1.html` — explicitly scoped as a "deployment & ownership view," not strict UML deployment (no per-service execution-node modeling), per the 2026-09-16 notation review.

## 8. Rollout

No implementation has started. This section has two parts: the **build order** (dependency-driven, what to write first) and the **launch plan** (how a built system actually goes live on real examinations) — a Google reviewer would expect both, and the build order alone is not a rollout plan.

### 8.1 Build order

1. Ingestion Adapter + Notice Corpus Manager (OCR/segmentation) + Fraud Database schema — nothing else can be built or tested against real data without these.
2. Intake Tracker (rule-based, no LLM dependency — lowest-risk first agent to ship).
3. Compliance Analyst + citation-grounding guardrail, behind the model-agnostic interface — the highest-stakes component, wants the most test coverage before any human sees its output.
4. Meeting Facilitator, Findings Author, Approval Coordinator, in phase order.
5. Examiner Dashboard, screen by screen, following the tab order already fixed by decision 88.
6. Observability/Langfuse wiring alongside agent work, not bolted on after (ADR-0016 assumes it exists from day one for the human-agreement-rate metric to mean anything).

### 8.2 Launch plan

- **Shadow mode first**: run the pipeline in parallel on 1–2 real, already-underway examinations without surfacing any agent output to the Lead/Approver as actionable — compare agent output against what the examiner team produced manually, offline. Purpose: get a real citation-validity and human-agreement-rate baseline before any examiner workflow depends on it.
- **Pilot phase**: a small number of full examinations (exact count TBD with FPSD — not yet decided, add to §9) run live through the pipeline with a Lead/Approver sign-off at every checkpoint as designed, but with an explicit fallback to the fully-manual process available at any point if the pipeline blocks progress.
- **Go/no-go gate out of the pilot**, tracked against concrete, code-checkable signals rather than a subjective call:
  - Zero verified instances of a compliance verdict citing a clause that doesn't exist or doesn't say what was cited (a hard gate — ADR-0004 is non-negotiable).
  - Human-agreement rate in a sane band, not near 0% (system is useless) or near 100% (automation-bias risk, ADR-0016) — exact band TBD from pilot data.
  - No `chain_integrity_violation` audit-log alarms unexplained by a known incident.
  - No SLA breach caused by a pipeline defect (as opposed to normal workload) going undetected until an examiner manually noticed.
- **Rollback trigger**: any Critical-severity defect found live (e.g., a fabricated citation reaching a human as if grounded, or a guardrail-bypassing state transition) halts new examinations from entering the pipeline and reverts in-flight examinations to manual handling for the affected step, without touching already-completed, signed-off findings. This is distinct from and in addition to ADR-0023's narrower canary/rollback process for *model swaps specifically* — that one assumes the pipeline itself is already trusted and is pinning in-flight examinations to a model version; this one covers the pipeline being wrong in the first place.
- **Full rollout**: only after the pilot's go/no-go gate passes and FPSD leadership records sign-off (see the **Approved by** field at the top of this document, and of `docs/PRD.md` — neither is yet filled in).

## 9. Open questions

See `docs/PRD.md` §11 for the full, current list (organizational sign-off/cost model, platform and model choice, IdP/secrets-manager product choice, checkpoint SLAs, capacity numbers, sanctions-screening and Notice-Corpus-management tool contracts, and the Arabic/bilingual confirmation). This TDD does not duplicate that list — check there first, and check `docs/decision-log.md` before assuming any of them have since been answered.

## 10. Reference map

Same as `docs/PRD.md` §13 — `docs/decision-log.md`, `CONTEXT.md`, `docs/architecture/agent-specifications.md`, `docs/architecture/tool-contracts.md`, `docs/architecture/` diagrams, `docs/reviews/`.
