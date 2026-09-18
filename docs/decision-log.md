# Decision Log — Questions Asked & Decisions Made

This is the durable, committed record of every question raised and decision made while designing the LFI Examination Pipeline — companion to `CONTEXT.md` (the distilled glossary) and `docs/adr/` (the distilled architectural decisions). Where `CONTEXT.md`/ADRs give you the settled *what*, this document gives you the *why*, in the order it was actually decided, including what was asked, what was considered, and what was rejected.

**Purpose**: so a question already settled here doesn't get silently re-litigated later. Before reopening something that looks settled, check here first — if new information genuinely changes the answer, the convention is to add a new round below (never edit history) and, where the decision is architecturally load-bearing, write a new ADR that explicitly supersedes/amends the old one.

**Relationship to other docs**:
- `CONTEXT.md` — canonical vocabulary, kept current, no history.
- `docs/adr/0001`–`0032` — one decision per file, the thing to cite in code/design reviews. Written in the Michael Nygard ADR format (Title/Status/Context/Decision/Consequences, as of 2026-09-19 — see `README.md`'s "ADR format" note).
- `.scratch/lfi-examination-pipeline/design-grill.md` — the original raw working log this document is distilled from (gitignored, not part of the shipped repo — this file is now the canonical, committed copy going forward).
- This file — the full narrative: every question, every round, every decision, in order, cross-referenced to ADRs.

Decisions are numbered sequentially (1–138 as of Round 22, 2026-09-18) and never renumbered; a decision that's later reversed or amended says so explicitly and points at the decision that changed it, rather than being edited away.

**Coverage gap noted 2026-09-19, backfilled same day**: fixes from the 2026-09-17 and 2026-09-18 Principal Engineer audits (`docs/reviews/`) had been applied directly to `agent-specifications.md`/`tool-contracts.md`/ADRs 0030–0032 without a corresponding decision-log round. Rounds 19–22 below now cover that ground. `docs/reviews/` remains worth reading directly for the full findings text (this file summarizes, cites commit hashes, but doesn't reproduce every finding verbatim).

---

## Original brief (verbatim)

FPSD (Fraud Prevention and Supervision Department) at CBUAE runs regular examinations of Licensed Financial Institutions (LFIs) for compliance against fraud, consumer protection, and market conduct standards. Three phases:

1. **Pre-examination**: RFIs (15-20 questions, each tagged to a specific notice/standard/clause) sent to LFI 30 days before start. LFI uploads documents (tables, flowcharts, screenshots, text) into a shared workspace, structured by RFI question, over the 30-day window. Team must track submitted vs pending before examination can start. Team then manually reviews every document against the cited notices/standards (tedious, error-prone), also cross-referencing quarterly fraud data submissions pulled from an EDM. Team drafts clarification questions for gaps/non-compliance.
2. **Examination meeting**: team asks clarification questions live. Meeting notes/minutes analyzed afterward for any further required submissions, which LFI submits via the same shared workspace mechanism.
3. **Post-examination**: findings finalized and validated → transmittal letter drafted (each finding gets severity High/Medium/Low + deadline) → pre-exit presentation deck drafted → both go to Assistant Governor for approval → presented to LFI leadership as pre-exit. Findings are very rarely revised at this stage; if they are, transmittal letter + deck are updated and re-shared as the **exit deck**.

Goal: convert this workflow into a multi-agent pipeline.

---

## Rounds 1–2 — foundational scope and constraints

**Question asked**: given the brief above, what are the automation philosophy, the deployment constraints, the source-system facts, and the hard regulatory rules this pipeline must respect?

1. **Automation philosophy**: Augmentation/co-pilot, not full autonomy. Human sign-off required at: (a) final list of clarification questions before the examination meeting, (b) final findings/severity before the transmittal letter, (c) before anything goes to the Assistant Governor. *(Later formalized as ADR-0003.)*
2. **Deployment/data constraints**: Must run in an approved/sandboxed enterprise environment (private cloud or on-prem LLM deployment) — LFI examination data is sensitive supervisory data. Hard constraint, assumed for design purposes pending IT/compliance confirmation.
3. **Source systems**: Shared workspace (LFI document uploads) — originally assumed no API/SFTP access, **superseded by decision 14** (SFTP is in fact enabled). EDM (quarterly fraud data) — Oracle, has an API, not a blocker.
4. **Citation requirement**: Any compliance verdict must cite the **exact notice and exact sentence verbatim** — no paraphrasing as if it were a quote. **Hard rule: no grounded citation → no compliance verdict.** If the agent can't find a high-confidence exact match, it outputs "insufficient grounding, needs human review" instead of guessing. *(Later formalized as ADR-0004 — the single most load-bearing rule in the whole design; see also decisions 56, 78.)*
5. **Cross-notice supersession**: Some notice clauses supersede clauses in other notices under certain conditions — a real dependency graph the pipeline must model. Agent drafts an initial supersession table from the notice corpus; only truly complex/high-uncertainty relationships route to human review. Escalation threshold mechanism settled in decision 30.
6. **Notice corpus readiness**: Mostly digitized PDFs; some older notices may be scanned images. OCR must be a supported feature, confirmed per-notice rather than assumed uniformly one way.
7. **Rollout scope for v1**: Originally narrowed to pre-exam intake tracking + gap analysis only — **reversed by decision 22**: v1 is a lean, bare-minimum, end-to-end pipeline covering all three phases.
8. **Agent orchestration tooling**: **LangGraph** — chosen for granular control, fits well with human-in-the-loop interrupt/checkpoint needs from decision 1. *(Later formalized as ADR-0001.)*
9. **Output deliverable formats**: RFI, transmittal letter, pre-exit deck, exit deck all follow fixed existing Word/PPT templates — agent fills them in, does not design new layouts from scratch.
10. **RFI-to-notice mapping**: Already exists — each RFI question is tagged with its corresponding notice/standard/law in the RFI itself. A given input, not something the pipeline needs to infer fresh each cycle.
11. **Severity/deadline rubric**: Confirmed to exist as a standard rubric (severity → deadline). Encoding settled in decision 16.
12. **Folder structure**: Fixed and standardized — follows the RFI's section/subsection structure, same for all LFIs of the same license type. This is what makes automated intake tracking tractable.
13. **RFI question bank**: RFIs are standardized per LFI license type (not hand-written per LFI each cycle), reinforcing decision 12.

## Round 3 — ingestion, rubric, and scope facts

**Question asked**: follow-up facts needed to make decisions 1–13 buildable — is SFTP actually available, is the RFI machine-readable, how is the rubric encoded, what license types are in v1?

14. **Ingestion mechanism confirmed**: Shared workspace has SFTP enabled — supersedes decision 3's "no API/SFTP access" assumption. Built as a pluggable adapter (list/fetch/get-last-modified interface); v1's concrete adapter is SFTP-based, a Smart-portal API adapter remains a future swap-in. *(Later formalized as ADR-0006.)*
15. **RFI machine-readability**: RFI is a structured Excel sheet (Question ID, Question text, Notice reference, Clause reference) — no free-text extraction/re-keying step needed. *(Revisited in Round 15/decision 71 — the Excel format is the ingestion source, not the storage engine.)*
16. **Severity/deadline rubric**: Directly coded as a lookup table, but must be modifiable from a UI (not a static config) — a small admin surface (CRUD on rubric entries). *(Revisited in Round 13/decision 61 — promoted to a shared, versioned platform service.)*
17. **v1 license type scope**: Banks only, confirmed. SVFs and other license types deferred to v2+.
18. **Definition of "submitted" (v1 scope)**: presence + non-empty + filename/format convention match only. Content/plausibility validation is a separate downstream agent, deferred to v2 (decision 28).
19. **LFI-facing communication**: Confirmed as a real future feature (e.g., reminder emails), but explicitly out of v1 scope.
20. **EDM cross-referencing dropped from v1**: EDM quarterly fraud data and LFI-submitted documents are non-overlapping datasets to a large extent — not enough overlap to justify document-vs-EDM consistency cross-checking in v1. *(Later formalized as ADR-0009.)*

## Round 4 — EDM's real role, and a scope reversal

**Question asked**: if EDM isn't used for cross-checking (decision 20), what *is* it for — and is the "intake + gap analysis only" v1 scope (decision 7) actually right?

21. **EDM integration stays in v1.** Role: EDM (Oracle) supplies LFI quarterly fraud submissions. Not used for automated consistency cross-checking (decision 20 stands). Instead it (a) complements document evidence as separate context during gap analysis, and (b) is a guiding input for drafting follow-up/clarification questions.
22. **⚠️ SCOPE REVERSAL of decision 7**: v1 is *not* limited to intake tracking + gap analysis. Transmittal letter drafting and pre-exit deck drafting are in v1 — v1 is a lean, bare-minimum, end-to-end pipeline covering all three phases. *(Cascading effect: decision 16's rubric-editing UI is back in v1 scope, since post-exam findings/severity assignment now needs it. Later formalized as ADR-0002.)*
23. **SFTP adapter — operational status**: Not yet provisioned, requires an IT/security request. Folder structure: SFTP will 1:1 mirror the shared workspace's fixed per-license-type structure (decision 12) — no separate mapping/translation layer needed.
24. **Content-validation agent scope**: Ambiguous pending Round 5 — resolved by decision 28.

## Round 5 — meeting phase and post-exam bare minimum

**Question asked**: given decision 22's scope reversal, what does "bare minimum" actually mean for the meeting phase and post-exam phase specifically?

25. **Meeting-phase bare minimum**: Agent drafts the pre-meeting clarification-question list from gap-analysis + EDM output; post-meeting, it accepts human-provided meeting minutes/notes as text input and extracts further-submission action items. No audio/video transcription in v1.
26. **Post-exam bare minimum**: Human sign-off checkpoint (decision 1) on findings/severity stays before transmittal-letter drafting — not skipped for "lean." Pre-exit deck is a derived artifact off the same findings data as the letter, not an independently-authored content path.
27. **AG approval — status gate only**: Pipeline tracks "drafted → awaiting AG approval → approved" as state; actual approval routing/notification is a manual human action outside the pipeline. Exit-deck revision path (rare edge case) deferred, handled manually if/when triggered. *(⚠️ Both halves of this decision were later reversed — see decisions 37, 38.)*
28. **Content-validation agent deferred to v2**: v1 ships presence/format checking only (decision 18); document-plausibility judgment stays a manual reviewer task for now.

## Round 6 — notice corpus and supersession mechanics

**Question asked**: how does the notice corpus actually get built and queried, and how does the supersession-escalation mechanism (decision 5) actually work?

29. **Notice corpus ingestion is v1 scope**: One-time (then incrementally-updated) ingestion step — PDF → text (OCR per decision 6 where needed) → chunked/indexed with per-clause addressability. *(Later formalized as ADR-0005.)*
30. **Supersession escalation**: Simple confidence-score cutoff for v1 (decision 5's threshold mechanism) — tunable later from real cases, no upfront complexity taxonomy needed. *(Revisited in Round 17/decision 78 — confirms the underlying storage architecture, doesn't change this threshold mechanism.)*
31. **Human review UX**: Per-feature review surfaces are acceptable for v1 (no unified review queue) — each flagged-item type surfaces in its own feature context; consolidation deferred to v2.
32. **Notice versioning**: v1 assumes current-version-only — no effective-dated historical clause versions. Explicitly flagged as a known v1 limitation. *(Revisited in Round 17/decision 78 — resolution path specified: structured relational columns, not a graph/RAG extension, when this is picked up.)*
33. **UI surface**: One unified app confirmed — clarified further in decision 34.

## Round 7 — UI scope, RBAC, and audit trail; frontier check

**Question asked**: what does "comprehensive UI" actually mean, and does v1 need role-based access control and an audit trail?

34. **"Comprehensive" UI clarified**: Means comprehensive *coverage of already-scoped touchpoints* — not additional feature scope beyond what's already decided. *(Later formalized as ADR-0008.)*
35. **Team structure / access control**: Lightweight two-tier RBAC confirmed for v1 — examiner role (drafts, works the flow) vs. lead/approver role (signs off at decision-1 checkpoints). *(Revisited in Round 15/decision 69 — a third, read-only Auditor/Compliance Reviewer role added.)*
36. **Audit trail is v1 scope**: Append-only log of agent-proposed vs. human-approved/overridden state at every sign-off checkpoint, built in from the start rather than retrofitted. *(Later formalized as ADR-0007.)*

**Frontier check**: all previously identified branches had settled decisions (1–36). User confirmed the frontier was empty on 2026-09-15; domain-modeling capture followed (`CONTEXT.md`, ADR-0001 through 0009) and the v1 architecture diagram was built and delivered.

## Round 8 — reconciling against the real current-state flowchart

**Question asked**: the user uploaded a flowchart of the actual current-state process (three phases + Data Office/ITD constraints + proposed data layers). It's richer than the original verbal brief in places, and contradicted two settled decisions — does the design need to change?

37. **⚠️ REVERSES part of decision 27**: The Phase-3 "LFI has concerns → update transmittal letter/deck → re-check → proceed to exit meeting" loop is **routine, not a rare edge case**. Now in v1 scope as a normal, modeled step. *(Later formalized as ADR-0010.)*
38. **⚠️ REVERSES part of decision 27**: The AG-review step is not a flat status field — the real process has an explicit revision loop (Showcase → Changes? → back to Apply Comments → re-showcase). v1 must model "AG requested changes" as real pipeline state that routes back into findings/letter/deck drafting. *(Later formalized as ADR-0010.)*
39. **Analysis timing confirmed as batch, not incremental**: Gap analysis is a single batch step triggered by complete intake, not a continuous per-document process during the 30-day window — formalizes what was already implicit.
40. **Transmittal Letter spelling confirmed correct** — the diagram's "Transmitter Letter" is a typo; "Transmittal Letter" stands.
41. **"Supervision Questions" adopted as the canonical term**, replacing "Clarification Question." "Clarification Question" becomes an alias to avoid in `CONTEXT.md`.
42. **Qualitative vs. Quantitative Analysis** distinguished as two first-class tracks (Phase 2's "Analyze findings"). Does *not* reverse decision 20/ADR-0009 — quantitative analysis is EDM data analyzed on its own terms, not reconciled against documents. *(Later formalized as ADR-0011; quantitative track given a real compute tool in decision 62.)*
43. **"Fraud Database" formalized as one umbrella data store**, containing Notice Corpus, Audit Trail Store, and gap-analysis/AI-generated output as logical sub-stores — conceptual separation in `CONTEXT.md`/diagrams stays, only the physical-storage framing changes. *(Later formalized as ADR-0012; extended in Round 15/decision 71 to explicitly include RFI Store as a fourth sub-store.)*

Also confirmed (no reversal, detail added): CB Workspace is being delisted, replaced by an internally-built Smart Portal reachable via API or SFTP.

## Round 9 — deployment-readiness facts

**Question asked**: user asked for a detailed, deployment-ready solution architecture with agents defined — what are the real infra facts (hosting, identity, database engine)?

44. **Deployment target: on-prem for now** (amended by decision 48). Originally "cloud-agnostic, not decided" — user then explicitly chose on-prem. Container-orchestration boundary stays generic (VMs vs. K8s/OpenShift still open).
45. **LLM hosting: self-hosted open-weight model, in-tenant.** No third-party model API call. Specific model left open pending real hardware capacity — resolved to "known candidates, still swappable" in Round 10. *(Later formalized as ADR-0013.)*
46. **Identity: federate to CBUAE's existing SSO** (Azure AD / on-prem AD) via OIDC/SAML — the Examiner/Lead-Approver RBAC roles (decision 35) are authorization claims on federated identity, not a new user store.
47. **Fraud Database engine: Oracle**, matching EDM's existing engine — chosen for CBUAE's existing Oracle operational expertise over pure workload fit. *(Later formalized as ADR-0014.)*

## Round 10 — on-prem confirmed, model candidates named

**Question asked**: is the deployment target actually locked to on-prem, and what specific model(s) are realistically available?

48. **Deployment target locked to on-prem** — not "cloud-agnostic, undecided" but an explicit choice, revisitable later. *(Formalized as ADR-0015.)*
49. **Model/GPU treated as a black box** — agents built against a model-agnostic interface. IT has two candidates available on-prem — **Qwen Coder** and **gpt-oss-120b** — named but neither committed to; stays a "revisit later" decision, not a default.

## Round 11 — observability, evaluation, audit, and HITL formalization

**Question asked**: user asked for audit/monitoring, observability metrics, evaluation metrics, and formalized HITL checkpoints, recommending industry best practice. (Also: the deployment diagram had collapsed all 5 agents into one box — fixed first, personas moved from a hidden-by-default field to an always-visible one.)

50. **Observability stack: OpenTelemetry + Prometheus/Grafana/Loki/Tempo** ("LGTM" stack) for infra-level traces/metrics/logs — RED method per service, USE method for the GPU-backed LLM Inference Service.
51. **LLM-specific tracing and evaluation: self-hosted Langfuse** — captures every agent's prompt/completion/tokens/cost linked to its OTel trace, hosts both automated and human-annotated evals on the same trace.
52. **Primary trust metric is human-agreement rate, not accuracy** — how often a Lead/Approver accepts an agent's draft unedited. A rate near 100% is flagged as an automation-bias risk, not a win.
53. **Audit consumption stays query-based against the Fraud Database** — no new audit-reporting service; read-only queries/reports against `audit_log`. Entries hash-chained for tamper-evidence.
54. **HITL checkpoints formalized as a table**: 5 checkpoints, each with a resolving role and a suggested SLA. `log_checkpoint()` records accept/edit/reject, not just resolved/unresolved. *(All four of 50–54 later formalized as ADR-0016; checkpoint table extended in Round 12/decisions 56–57 to 7 checkpoints.)*

## Round 12 — independent architecture review + Critical-finding remediation

**Question asked**: user asked for an independent senior-engineer-level review of the entire architecture, deliberately spun up as a fresh subagent (not a fork) to avoid the review being anchored to this session's own reasoning. Verdict (`.scratch/architecture-review-2026-09-15.md`): "every artifact stops at the decision layer and never reaches the contract layer" — 4 Critical, 6 High, 5 Medium, 2 Low findings. User chose to work through all 4 Critical findings first.

55. **LangGraph state-machine diagrams built, one per phase** — fixes Critical Finding #2 (no node/edge/conditional-transition diagram existed). Split by phase because archify's workflow diagrams cap at 6 logical columns per lane.
56. **Guardrail enforcement generalized into a pattern, not a one-off** (ADR-0017) — fixes Critical Finding #1. Findings Author's persisted `severity` must equal `severity_rubric.lookup()`'s actual return value, checked in code; AG-summarized feedback no longer feeds a redraft trigger directly, it surfaces first as its own reviewable artifact behind a new "AG Feedback Review" checkpoint.
57. **Sufficiency-check gate added to Phase 2** — fixes Critical Finding #3. A real, human-judged gate between further-submission receipt and Findings Author's trigger — the same category of error ADR-0010 already caught once for the AG/pre-exit loops.
58. **Failure, retry, and idempotency semantics defined** (ADR-0018) — fixes Critical Finding #4. Tool-call retry with backoff (tool-unavailable ≠ no-match), idempotency keys on state-mutating calls, optimistic concurrency on `interrupt()` resolution, schema-validation-with-repair on structured output.

## Round 13 — all 6 High findings fixed

**Question asked**: user said "lets go" to proceed straight through the review's High-severity findings.

59. **Tool contracts written as real schemas** (`tool-contracts.md`) — every tool now has input/output schema (distinct error shape, never collapsed into "no result found"), timeout, retry policy.
60. **Threat model + network isolation for the LLM Inference Service** (ADR-0019) — dedicated security-group boundary, mTLS-only access from the 4 LLM agents, encryption at rest on the Fraud Database, a new `secrets_manager` component.
61. **`severity_rubric.lookup` promoted to a shared, versioned platform service** (ADR-0020) — every finding carries the `rubric_version` it was evaluated against; a redraft re-evaluates against the current version, with drift surfaced explicitly, not applied silently.
62. **`quant_analysis.compute()` deterministic tool added** (ADR-0021) — quantitative arithmetic is now code, never the LLM computing directly from raw EDM figures in-context.
63. **Capacity/scale placeholder plan** (ADR-0022) — explicit, illustrative-only GPU/storage numbers, marked for revisit once real FPSD usage data exists.
64. **Testing, canary, and rollback strategy** (ADR-0023) — unit tests for tool error paths, a LangGraph integration suite, a canary process for model swaps, and an in-flight-examination pinning policy.

## Round 14 — logical diagram redesigned for phase/role clarity

**Question asked**: user asked to clearly mark the 3 phases, add human-performed steps as distinct nodes colored differently from automated ones, and explicitly separate "LLM agent," "LLM agent + tool call," and "deterministic tool" into different colors — aligned to the real current-state flowchart.

65. **Phase boundary labels renamed to match the flowchart verbatim**: "Phase 1: Sending out the RFIs", "Phase 2: Examination Phase (Onsite)", "Phase 3: Pre-Exit Phase."
66. **Human-performed steps added as first-class nodes**: LFI Kickoff Comms, In-Person Deep Dive, AG Showcase + Pre-Exit — marked as human, not modeled as pipeline state.
67. **Component color now encodes automation role, not just architecture layer** — a custom legend remaps archify's fixed component-type colors: grey = human/external, teal = LLM-reasoning-only, peach = LLM+tool, gold = deterministic, violet = data store, blue = human-facing UI. Disclosed limitation: gold/peach are close in hue.
68. **Per-phase background tinting from the flowchart was not replicated** — archify's region boundary has no per-instance custom color; matched what the flowchart communicates (phase separation + labels), not its literal color choice.

## Round 15 — all 5 Medium findings fixed (2026-09-16)

**Question asked**: user said "yes, lets go" to proceed through the review's Medium-severity findings.

69. **Data retention, lifecycle, and audit-access review** (ADR-0024) — Langfuse traces get bounded retention shorter than the Fraud Database's audit trail; a new read-only **Auditor/Compliance Reviewer** RBAC role added; voided examinations flagged, never physically deleted.
70. **LangGraph state schema versioning** (ADR-0025) — explicit `schema_version` field on the state object; any breaking change requires a tested migration function before shipping.
71. **RFI Store confirmed as the Fraud Database's fourth logical sub-store** (extends ADR-0012) — decision 15's "structured Excel sheet" was the ingestion source format, not the storage engine.
72. **`emerald` card-dot color now reserved exclusively for "deferred/out of scope"** across both architecture diagrams — no longer means opposite things in the two diagrams meant to be read together.
73. **"5 agents" framing corrected to "4 LLM agents + 1 rule-based service"** everywhere — self-consistent with Intake Tracker's own "not an LLM" caveat.

Only the review's 2 Low findings (cost model, independent model-risk/compliance sign-off process) remained open after this round — both organizational, not architectural.

## Round 16 — diagram naming/notation audit + agent renaming (2026-09-16)

**Question asked**: two separate requests. First, clear out unnecessary architecture files with clear naming/versioning. Second, audit the diagrams against industry-standard notation (C4/UML/BPMN), spun up as an independent solution-architect review (fresh agent, to avoid anchoring).

74. **Diagram filenames standardized to `<subject>-v1.<ext>`** — no files were actually unused (all 5 pre-existing diagrams were live and referenced); the fix was naming consistency, since the 3 workflow diagrams lacked the `-v1` suffix the 2 architecture diagrams had.
75. **Diagram notation review verdict: mostly sound given archify's actual grammar.** One real cross-diagram bug fixed — the deployment diagram's colors had silently reverted to default type semantics instead of carrying the logical diagram's automation-role legend, misleadingly coloring rule-based Intake Tracker the same as the real LLM agents; component types retyped to match actual automation role. Two cheap missing artifacts added: `lfi-pipeline-context-v1` (a real C4 Context diagram) and `lfi-guardrail-citation-grounding-v1` (a UML sequence diagram for the citation-grounding retry-vs-real-absence call flow). "How to read this diagram" notation footnotes added to the 3 workflow diagrams (archify's workflow grammar has no decision-diamond/fork-join primitive — a tool constraint, not sloppy authoring) and a scope caveat added to the deployment diagram (a "deployment & ownership view," not strict UML deployment).
76. **Agents renamed to their professional persona titles** — user confirmed scope explicitly (headline/display names only, not internal ids): Gap Analysis Agent → **Compliance Analyst**, Clarification & Meeting → **Meeting Facilitator**, Findings & Reporting → **Findings Author**, AG & Pre-Exit → **Approval Coordinator**. All 4 persona names already existed in `agent-specifications.md`'s Persona lines — promoted to the reader-facing name everywhere, old functional name preserved in each diagram's sublabel/tag.

**Incident worth remembering**: the rename was first delegated to a background fork; it ran 31 minutes with no completion notification (vs. 1.5–4.5 min for three parallel research forks in the same batch) — almost certainly stuck cycling on archify's label-width/desktop-readability validator, the same fiddly category of fix the main thread had already hit manually earlier in this project. User chose to abandon and redo directly; the fork was cleanly stopped, its one fully-correct completed file was kept (no worktree isolation, so nothing was lost), and the remainder finished directly. Lesson: don't delegate a task mixing mechanical text edits with archify diagram-validator loops to an unsupervised fork — the validator portion needs interactive judgment a fork can't escalate out of.

## Round 17 — three decisions from user-directed research (2026-09-16)

**Questions asked**: user raised five points in one message: (1) how do we evaluate whether an agent's work is "done," and should we borrow ideas from Google's DS-STAR paper (Planner/Verifier/Router) to add a Verifier agent; (2) professional agent naming (handled directly, see Round 16/decision 76); (3) explicitly define success/completion criteria at each step; (4) what's the best strategy for storing the notices/laws/standards agents cite against — RAG, a structured table, or a knowledge graph; (5) a separate sub-agent using web tools/MCP to enrich examinations with fraud news, consumer complaints, etc. Points 1/3/4/5 were routed to independent research forks (each given full project context) before being turned into ADRs.

77. **No dedicated Verifier agent** (ADR-0026) — DS-STAR (confirmed: arXiv 2509.21825, Google Research, Sept 2025 — Planner/Coder/Verifier/Router, capped at 10 rounds) doesn't port cleanly: its Verifier judges a runnable, checkable artifact (code executed against data); this pipeline's outputs are human regulatory judgments by design (ADR-0003). A literal port — an LLM Verifier approving output and unblocking autonomous continuation — would conflict with augmentation-not-autonomy directly. Decision: the existing code guardrails (ADR-0017/0020/0021) plus HITL checkpoints already implement the same underlying idea, better-fitted to a domain where the real arbiter must stay human. A narrow, flag-only draft-completeness pre-check before Findings Author's sign-off is left as a future option, not built.
78. **Explicit per-agent "definition of done"** added to `agent-specifications.md` (part of ADR-0026) — concrete, checkable completion predicates per agent, distilled from guardrails/state fields that already existed but were scattered across several ADRs and the shared-state table.
79. **Notice/clause corpus storage strategy confirmed** (ADR-0027) — the existing hybrid (`notice_corpus.search()` for discovery + `notice_corpus.get_clause()` for exact grounding + `supersession_graph.check()` for single-hop relationships) is already correct for this domain. External research: legal-citation RAG hallucinates 17–33% of the time, and "hallucination with a citation is worse than plain hallucination" — directly validating ADR-0004's exact-match-or-refuse rule. `supersession_graph` is a lookup table wearing a graph-shaped API, which is fine since nothing needs multi-hop traversal. Decision 32's deferred notice-versioning now has an explicit resolution path: structured `effective_from`/`effective_to` columns when picked up, not a graph/RAG extension.
80. **External-intelligence enrichment split into two proposals** (ADR-0028) — cuts against the same conservative line ADR-0009/0011 already drew once for EDM. Split: **sanctions/watchlist screening against official government lists** (OFAC/UN/EU/UK) scoped as a near-term, lower-risk addition — real MCP servers exist, structurally citation-grade like a notice clause. **News/adverse-media/consumer-complaint monitoring** explicitly deferred, pending an actual answer from CBUAE legal/compliance on whether FPSD is even authorized to build this at all — a legal question, not an engineering one, given known false-positive rates (35–95%) and real ToS/personal-data exposure. If ever built, must surface as a separate, clearly-labeled advisory signal, never blended into a grounded citation.

## Round 18 — Examiner Dashboard userflow (2026-09-17)

**Question asked**: user asked to work through the userflow — the screen-by-screen journey an Examiner, Lead/Approver, and Auditor actually take through the single unified app (ADR-0008) across all three phases — expecting it to surface further architectural decisions. Grilled across nine rounds covering navigation/portfolio structure, per-tab screen content and RBAC-gated actions for every HITL checkpoint, and data/admin screens (LFI registry, Notice Corpus, rubric editor, login).

**Navigation and portfolio structure**
81. **Portfolio home view**: landing page lists all active examinations (LFI, license type, current phase, SLA countdown/breach flag, open-checkpoint count, last-activity timestamp), filterable Active/Completed/All (default Active) — what a Lead/Approver sees before entering any single examination's workspace.
82. **Examination creation is a one-time in-app setup screen, not automated RFI generation**: Lead/Approver picks an LFI, sets a start date and SFTP root path, and **uploads the RFI already sent to the LFI** — the app never generates or sends an RFI itself. Reverses an earlier assumption that the RFI would be auto-generated from the license-type template.
83. **Navigation model: tabs, not a phase-stepper**, behind a permanent sidebar (user/role details anchored at the bottom). Tabs unlock by data availability, not a forward-only phase gate, since earlier-phase artifacts stay live reference material later and the AG/pre-exit revision loops (ADR-0010) aren't strictly linear.
84. **Severity/deadline rubric editor is global** (per license type, not per-examination), edited from a standalone Settings/Admin area — consistent with ADR-0020's "shared, versioned platform service" framing. Access: Lead/Approver; no new admin role.
85. **RFI upload auto-parses into the expected question list**, shown back as an editable table to confirm/correct before setup finalizes — insurance against a hand-edited RFI drifting from the standard spreadsheet template.
86. **SFTP path is one root path per LFI examination cycle** — the fixed per-license-type folder structure (decision 12) is expected underneath it, so setup needs no manual folder-mapping step.
87. **Transport choice (Shared Workspace vs. future Smart Portal) is platform-wide config, not a per-examination field** — a transport picker on every setup screen would leak the Ingestion Adapter's abstraction (ADR-0006) back into the UI.
88. **Sidebar tab list, finalized**: Setup → Intake → Gap Analysis → Meeting → Findings → Approval → Audit Log. Kept separate despite the supervision-questions bridge between Gap Analysis and Meeting, since merging would obscure the sufficiency-check gate's "new round-trip" nature.

**Access, data ownership, and portfolio detail**
89. **RBAC visibility model: one screen per tab, gated actions — not parallel UIs per role.** Examiner and Lead/Approver see identical data/layout; sign-off controls render only for Lead/Approver.
90. **This app owns a lightweight LFI registry** (name, license type, SFTP root, contacts) rather than integrating against an unnamed external FPSD system of record — none has been named anywhere in the decision log.
91. **Notifications are in-app only for v1** — SLA breaches/backlog surface via the portfolio home screen's counts (agent-specifications.md); no email/notification infrastructure in v1.
92. **Portfolio home columns finalized**: LFI name, license type, current phase, SLA countdown/breach flag, open-checkpoint count, last-activity timestamp, plus decision 81's status filter.

**Intake and Gap Analysis tabs**
93. **Intake tab supports a manual override** (Lead/Approver only, required free-text reason) to mark a `suspicious`/`pending` item `submitted` by hand — logged as a distinct event type from the automated status.
94. **Document access is download/open via the Ingestion Adapter, not a custom in-app viewer.**
95. **Gap Analysis resolution actions, finalized as two**: (a) accept as `insufficient_grounding` (becomes a supervision question), (b) manually assert a verdict with a human-supplied citation (human-asserted, not model-asserted). A third option — targeted re-run of Compliance Analyst on a single item — was considered and dropped for v1; nothing in the LangGraph shape supports partial re-invocation from the UI.
96. **"Gap analysis complete and examiner-reviewed" (Meeting Facilitator's trigger, agent-specifications.md) is automatically inferred**, not a separate sign-off button — true once every flagged item has a decision-95 resolution and every RFI question has a `compliance_verdicts[]` entry.

**Meeting tab**
97. **Supervision-question sign-off is per-question**, not whole-list — matches the human-agreement-rate metric's per-item design (ADR-0016).
98. **Meeting minutes are submitted after the fact, one-shot** — not typed live during the meeting.
99. **New HITL checkpoint added: Further-submission review.** `further_submission_requests[]` (extracted from meeting minutes) previously had no checkpoint at all — a real gap, since the supervision-question sign-off happens *before* the meeting and covers a different artifact. Raised by Meeting Facilitator, resolved by Lead/Approver, per-item accept/edit/reject like decision 97, before the list is final. Added as a new row to `agent-specifications.md`'s HITL table.
100. **LFI communication of further-submission requests stays entirely out of the app's scope** — same as the RFI (decision 82), the app never messages the LFI; the examiner relays the list manually. **Automated intake tracking for round-2 documents is explicitly kept** — user's clarification: whatever the LFI submits back, even multiple documents, still flows through the same Intake Tracker mechanism as round-1 documents, so a second round's volume doesn't cost examiners manual-tracking time. Decision 99's checkpoint and the auto-extraction both stand unchanged.

**Findings and Approval tabs**
101. **Findings sign-off is per-finding**, with qualitative and quantitative tracks in visually separate sections — reinforces ADR-0009/0011's rule that the two tracks are never reconciled against each other.
102. **Transmittal letter / pre-exit deck stay download-only in v1** (no in-browser rendering); the sign-off checkpoint covers `findings[]` data, not the rendered template. **In-app rendering is an explicit deferred v2 candidate, not discarded** — user asked this be kept as future scope.
103. **AG status reported via dropdown + required free-text on `changes_requested`** — the raw text is the input to the LLM-summarization-into-redraft-notes step (agent-specifications.md, Critical Finding #1).
104. **AG Feedback Review checkpoint shows raw feedback and the AI summary side-by-side**, with the summary (not the raw text) editable — showing only the summary would defeat the guardrail's purpose.
105. **Pre-exit concerns loop reuses the Findings tab's redraft/sign-off flow** — no separate "Exit Deck" tab; the exit deck (CONTEXT.md) is the same artifact relabeled once produced via this loop.

**Audit, lifecycle, and cross-cutting**
106. **Audit Log tab shows identical full detail to all three roles within one examination** — the Auditor distinction is scope, not detail: Examiner/Lead-Approver reach it only within an assigned examination, while Auditor gets a portfolio-wide, cross-examination Audit Log view as their effective landing page.
107. **Examination auto-closes (Active → Completed) the instant `preexit_status` reaches `proceeded_to_exit`** — no manual "Close Examination" button; the state is already an unambiguous, code-checkable terminal condition (ADR-0026's definition-of-done pattern).
108. **Notice Corpus management moves fully in-app for v1** — reverses the round's initial recommendation (offline/ops ingestion). User's reasoning: no one will maintain the corpus outside the app, so it must be a real in-app capability.
109. **Two-role RBAC (Examiner vs. Lead/Approver, decision 35) reconsidered and reaffirmed, unchanged.** User raised whether one role would do; recommendation given and unopposed: keep two — the augmentation-not-autonomy design (ADR-0003), every HITL checkpoint, ADR-0017's and ADR-0020's guardrails, and the human-agreement-rate metric (ADR-0016) all assume the drafter and approver are different people. Team-size staffing realities (one person holding both role-grants) don't require collapsing the concept itself.
110. **Rubric editor is a simple current-state edit form**, not a version-history browser — the data stays versioned (ADR-0020), but the editor just shows "current version: vN" plus save-creates-new-version; full history stays queryable via the Audit Log tab.
111. **Setup tab fields are editable after examination creation** (Lead/Approver only), not write-once — logged to `audit_log[]` as a distinct "setup corrected" event, same treatment as decision 93.
112. **Login/authentication assumed to be existing CBUAE SSO** — this app consumes an authenticated session + role claim, no local user/role management screen. Flagged as an open item (which specific IdP), same treatment as ADR-0019's `secrets_manager` and the deployment platform (ADR-0015).
113. **Notice ingestion auto-runs OCR + auto-segments into clauses**, shown for human review/correction before anything becomes citable — same pattern as decision 85, required by ADR-0004's zero-tolerance grounding rule.
114. **Notice Corpus management access is Lead/Approver, no new Compliance/Legal Content Owner role** — same reasoning as decision 84.
115. **Supersession-graph review stays reactive, inline within Gap Analysis** (decision 95's `needs_human_review` path) — no standalone corpus-wide supersession browser in v1.
116. **In-app metrics/observability dashboard stays out of v1** — ADR-0016 already decided audit/metrics consumption happens via Langfuse/BI-tool queries, specifically to avoid a second reporting surface. **Flagged as an explicit future-scope candidate** — a single unified metrics view may be worth adding in a later version, per user.
117. **Tool-failure surfacing: a visible "sync degraded" banner only after sustained failure** (e.g. 3 consecutive missed polling intervals), not on every transient retry (ADR-0018's backoff already handles those silently).

New ADR: **ADR-0029** captures this round's structural decisions (portfolio + tabbed workspace, RBAC-gated single-screen model, setup-is-upload-not-generation, Notice Corpus brought in-app, new Further-submission-review checkpoint). Per-screen prose detail lives in `agent-specifications.md`'s Examiner Dashboard section and the updated HITL table, not a second document.

## Round 19 — Examiner Dashboard diagram build-out + two independent coverage reviews (2026-09-17)

**Question asked**: with ADR-0029 settled, build the full diagram set for the Examiner Dashboard, then independently check whether every diagram actually draws what the decisions/ADRs say it should (a coverage check, not a full re-audit).

118. **11 Examiner Dashboard diagrams built** (commits `5a2cd1a`, `e209380`, `0ce4087`, `1c0cc3f`): Notice Corpus Manager added to the logical pipeline diagram; per-tab workflow diagrams for Setup, Intake, Gap Analysis, Audit Log, Rubric Editor, Further-submission Review, AG Feedback Review, and Findings Sign-off; plus the overall Examiner Dashboard userflow diagram.
119. **First coverage review's 2 High findings fixed** (commit `60e5092`): the `redraft` state given a card note where a prior diagram had left it implicit; the parallel-UI-per-role shape (contradicting decision 89's single-screen model) collapsed into one `Gated Actions` node.
120. **First coverage review's Medium/Low findings fixed** (commit `580d60e`): a missing post-creation Setup edit segment added; a dedicated `lfi-supervision-signoff-v1` diagram created (previously undrawn); tab-unlock-by-data-availability (decision 83) tagged explicitly where it had been implicit.
121. **Diagram gallery published to GitHub Pages / Cloudflare Pages** (`docs.codegraphlabs.com`, commit `e949b71`) — the diagram set is now browsable outside the repo, not just as raw HTML files.

## Round 20 — Principal Engineer MECE audit (2026-09-17, third independent review) and its blocking-condition fixes

**Question asked**: an independent, deliberately-unanchored Principal Engineer review audited the full corpus (`CONTEXT.md`, all 29 ADRs at the time, `agent-specifications.md`, `tool-contracts.md`, all 19 diagrams) across 6 dimensions — architecture correctness, workflow/state-machine correctness, sequence correctness, cross-artifact consistency, non-functional readiness, and build-readiness. Full report: `docs/reviews/2026-09-17-principal-engineer-audit.md`. Verdict: **"Yes, with conditions"** — sound architecture, but 6 blocking conditions had to close before the code they govern could be written, principally because "a diagram or ADR was running ahead of the build-level artifact underneath it" (the review's own naming of the pattern across all 3 reviews to date).

122. **ADR-0003's third checkpoint made real** (commit `f34e722`): ADR-0003 always named three mandatory sign-offs, but the canonical HITL table had no pre-AG-submission row, so nothing before this fix stopped a drafted transmittal letter/pre-exit deck reaching the Assistant Governor without a human ever reviewing the rendered document itself. Added **Pre-AG Sign-off** as HITL checkpoint 6 (Findings Author raises, Lead/Approver resolves, gates `ag_status` advancing to `awaiting_ag`) and a corresponding node in the Phase-3 LangGraph diagram. This is the single highest-consequence fix in this round — the review's finding 2.1, rated Critical.
123. **Both AG-feedback and pre-exit revision loops drawn as real state, and routed through re-signoff** (also commit `f34e722`) — ADR-0010 already said these loops were routine pipeline state, not a rare edge case, but no LangGraph diagram actually drew them; `lfi-workflow-phase3-postexam-v1` was missing outgoing edges from `ag_feedback_review` entirely. Fixed so that **any** redraft (from either loop) re-enters at `draft_findings` and must pass **both** Findings/severity sign-off and Pre-AG Sign-off again before `ag_status` can reach `awaiting_ag` a second time — closing a path where a redraft could silently change a finding's severity and reach the AG unreviewed (review's finding 2.3, its single highest-consequence workflow defect).
124. **`audit_log[]` record schema and `log_checkpoint()`'s contract fully specified** (commit `ca3f4ce`) — previously the audit trail, the system's designated regulatory source of truth (ADR-0007), had six different load-bearing requirements scattered across ADRs (hash chain, idempotency key, actor, distinct event types, accept/edit/reject + diff, distinct reason codes) and no single schema defining a field. Now specified in `tool-contracts.md`'s `log_checkpoint()` contract and the shared state table. Rated Critical by the review specifically because a hash chain cannot be retrofitted onto an existing log after the fact.
125. **Shared state table extended with the missing examination-record fields** (also commit `ca3f4ce`) — the canonical state table was missing the identity/lifecycle/ownership/scheduling fields the Examiner Dashboard's own spec and ADR-0029 already assumed existed: `lfi_id`, `license_type`, `phase`/`examination_status`, assignment, `voided` (ADR-0024's voiding policy had no field to set), `model_version` (ADR-0023's in-flight pinning had no field to pin), and `rfi_questions[]` (previously only `rfi_responses[]` existed, with nothing to check responses against). Rated Critical — the review called this "the single largest build-readiness gap."
126. **`severity` enum collision resolved**: `CONTEXT.md` defined High/Medium/Low (3 values); `tool-contracts.md`'s `severity_rubric.lookup()` returned a 4-value set including `"critical"`. Resolved in favor of `CONTEXT.md`'s canonical 3-value High/Medium/Low, lowercase wire form — `tool-contracts.md` corrected to match, since this field is checked by a code-enforced equality guardrail (ADR-0017) that would otherwise fail unpredictably. Rated Critical (review's finding 4.1).
127. **`all_submitted` vs. `round_submitted(N)` disambiguated, closing the round-2 intake mechanism gap** (also commit `ca3f4ce`) — `agent-specifications.md` had two mutually exclusive definitions of `all_submitted` in the same document, and further-submission (round-2) documents had no defined path onto Intake Tracker's tracking mechanism at all, since they aren't RFI questions. Fixed: `all_submitted` fires only on round-1 RFI questions reaching `submitted` (Compliance Analyst's Gap Analysis trigger, unchanged); a distinct `round_submitted(round: N)` event fires when all round-N further-submission rows are `submitted`, feeding Meeting Facilitator's sufficiency check specifically and never re-triggering Gap Analysis. Rated Critical (review's finding 2.6/2.7).
128. **Consistency sweep** (commit `42c0609`): "clarification question" (used in 5 ADRs despite `CONTEXT.md` explicitly listing it as a forbidden synonym of "supervision question") replaced throughout; RBAC role count fixed to a consistent three (Examiner / Lead-Approver / Auditor) everywhere, resolving ADR-0007's title still saying "two-tier" against ADR-0024's later third role; `severity_rubric.lookup()` given a `license_type` parameter (previously unreachable per-license-type, which would have shipped silently wrong once SVF license types entered scope); the rubric version-history access route fixed so the Lead/Approver who edits the rubric — not just the read-only Auditor — has a path to see its history.
129. **Three new ADRs written to close the review's non-blocking "written position" requests** (commit `14b2665`): **ADR-0030** (prompt injection via LFI-authored content — accepted, bounded risk, not a silent gap), **ADR-0031** (v1 is English-only, an explicit decision pending FPSD confirmation, not an unexamined assumption), **ADR-0032** (Fraud Database stores full document content, not pointers, because the source Shared Workspace is being delisted). Tool contracts also filled in for three previously-uncontracted tools sitting on the critical path: the RFI parser, the OCR/segmenter, and `ingestion_adapter.fetch_document()`.

**Not fixed this round, explicitly carried forward as non-blocking** (the review's Dimension 1 architecture-diagram findings): the deployment diagram still has no `ingestion_adapter` component (SFTP drawn straight into Intake Tracker); `severity_rubric`, `quant_analysis`, and `template_fill` — all three fully tool-contracted — still appear in no architecture diagram; Notice Corpus Manager is still mistyped/absent on the deployment diagram; several deployment-diagram edges (LLM inference, DB persistence, observability) are drawn as one representative edge with no "not exhaustive" annotation; Approval Coordinator is still typed as a pure deterministic tool despite carrying a guarded LLM call; the Context diagram still draws notice ingestion as a system feed (contradicting ADR-0029's human-upload model) and still omits the Auditor actor. These were explicitly triaged as cheap-but-not-blocking and left on the board.

## Round 21 — Seven additional sequence diagrams closing a coverage gap the user flagged directly (2026-09-18)

**Question asked**: user said "we need a lot more sequence diagrams, these are not enough" — at the time only 3 of 18 diagrams were true participant/message sequence diagrams (Setup Tab, Notice Corpus Manager, Citation-Grounding Guardrail); every other Examiner Dashboard tab only had a lane/phase workflow view. Scoped via AskUserQuestion to "fill in the remaining dashboard tabs" rather than assumed.

130. **7 new sequence diagrams added** (commit `af0f5c2`): Intake Tab, Gap Analysis Resolution, Supervision-Question Sign-off, AG Feedback Review, Further-Submission Review, Findings Sign-off, Audit Log scope, and the Severity Rubric Editor — all `<name>-sequence-v1`, every write routed through `log_checkpoint()` per `tool-contracts.md`. All 7 passed archify's showcase validation on the first candidate. Gallery restructured into lifecycle order (System Architecture → Dashboard orientation → Phase 1 → 2 → 3 → cross-cutting/admin) to keep workflow/sequence diagram pairs adjacent (commit `b473c85`).

## Round 22 — Principal Engineer audit, fourth pass (2026-09-18), and its fixes

**Question asked**: user asked for a fresh, deliberately-unanchored Principal Engineer review of **all** diagrams (not just the 7 new sequence ones), plus industry-benchmark research (C4, UML sequence combined fragments, arc42, HITL/approval-gated-AI documentation practice) and an explicit check that every diagram is self-explanatory. Full report: `docs/reviews/2026-09-18-principal-engineer-audit.md`. Verdict: **"not yet fully implementation-ready, but materially closer... the gap that remains is narrow and well-characterized."** All 6 of Round 20's blocking conditions were independently re-verified as genuinely fixed (not just narrated). Industry-benchmark comparison found the corpus ahead of typical practice for this stage (unprompted ADRs for injection/language, a hash-chained audit log designed pre-code); the C4 Context+Container pairing and arc42 mapping were both judged appropriate, with only the quality-requirements-as-scenario-table and a consolidated risk register named as genuinely thin.

131. **AG-feedback redraft sequence diagram fixed to show the mandatory re-signoff** (commit `61a5674`) — `lfi-ag-feedback-review-sequence-v1` had jumped straight from "redraft complete" to `awaiting_ag`, omitting the two re-signoff messages that `lfi-workflow-phase3-postexam-v1` (fixed in decision 123) already drew correctly — a builder following the sequence diagram alone would have recreated the exact defect decision 123 closed. Rated High.
132. **Phase 2 workflow diagram's stale card text corrected** (also commit `61a5674`) — its card still asserted the further-submission loop re-enters via `all_submitted`, the exact wrong trigger decision 127 replaced with `round_submitted(N)`; a diagram built in the very same batch (`lfi-further-submission-review-sequence-v1`) already had it right. One-line fix to match. Rated High.
133. **`interrupt()` resolution's optimistic-concurrency/etag protocol (ADR-0018 §3) given a drawn conflict branch** (also commit `61a5674`) — previously named only in ADR-0018's prose, with no diagram anywhere showing the etag being issued, carried, or checked, despite being the one mechanism common to every checkpoint in the system. Added to the Gap Analysis Resolution sequence diagram. Rated High.
134. **Deployment diagram fixed**: `ingestion_adapter` added as a real component; a merged "Shared Platform Tools" component added covering `severity_rubric`/`quant_analysis`/`template_fill` (previously undiagrammed despite full tool contracts); Notice Corpus Manager added. Closes Round 20's carried-forward Dimension-1 findings.
135. **`chain_integrity_violation` given a drawn branch** on the Audit Log sequence diagram (previously happy-path only, despite being the one failure mode `tool-contracts.md` says must halt and page rather than retry).
136. **Approval Coordinator retyped** `messagebus` (LLM + tool call) instead of `cloud` (deterministic, "no LLM involved at all" per the diagrams' own legend) in both architecture diagrams — it carries a guarded LLM summarization call (ADR-0017) and was mistyped since Round 14's automation-role color taxonomy was introduced.
137. **Context diagram fixed**: the notice-ingestion edge no longer implies a live system-to-system feed (ADR-0029 established human upload, not a publishing system); the missing Auditor/Compliance Reviewer actor (standing read access under ADR-0024) added to the actor set.
138. **New finding not raised by either prior review, fixed anyway**: an optional `model_confidence` field added to `ComplianceVerdict`/`FindingRecord` — `supersession_graph.check()` already computes a confidence score that gates a routing decision, but the number itself was being discarded rather than persisted onto the record it gated, which the review flagged as a real (if cheap) gap for a regulator that may someday need to reconstruct how confident the system was when a human accepted a verdict.

**Not fixed, explicitly named as secondary polish, not a backlog**: four diagrams flagged for "add one more card sentence" (none needing a rewrite) — `lfi-findings-signoff-v1` (why qualitative/quantitative tracks are separate lanes), `lfi-examiner-dashboard-userflow-v1` (a "where to go next" navigation card), `lfi-pipeline-context-v1` (flag the ADR-0029 tension on its own notice-ingestion card, not just fix the edge), `lfi-audit-log-scope-sequence-v1` (name `chain_integrity_violation` explicitly even where a full branch isn't drawn). Also unresolved: no confirmed drawn retry/timeout/exhaustion branch on 8 of the new sequence diagrams (a disclosed, defensible scoping choice, not a defect); `lfi-gap-analysis-resolution-sequence-v1`'s `asserted_by` value left implicit on one branch (Low).

---

## Open items (not yet decided)

- The review's 2 Low findings: a cost model, and an independent model-risk/compliance sign-off process for the AI system itself — both organizational, not architectural (see `.scratch/architecture-review-2026-09-15.md`).
- A Fraud Database ER diagram — **the lightweight PRD half of this item is now done** (`docs/PRD.md` + `docs/TDD.md`, 2026-09-19); the ER diagram remains outstanding (archify has no ER-capable diagram type; needs a different tool, e.g. Mermaid `erDiagram`).
- Decision 77's optional draft-completeness pre-check — not built; revisit only if Findings Author drafts are observed wasting reviewer time on completeness issues specifically.
- Round 20's carried-forward Dimension-1 diagram findings (decision 122's block, resolved in Round 22 per decisions 134/136/137) — cross-check `docs/reviews/2026-09-17-principal-engineer-audit.md` §"Dimension 1" against the current diagrams if reopening this area; most were closed by Round 22, but re-verify rather than assume.
- Round 22's secondary-polish items: 4 diagrams needing one added card sentence each, no drawn retry/timeout branch on 8 sequence diagrams (disclosed scoping choice, not a defect), and `lfi-gap-analysis-resolution-sequence-v1`'s implicit `asserted_by` value on one branch — all Low/cosmetic, listed in full at the end of Round 22 above.
- Decision 80's sanctions-screening tool — scoped as a direction, but no tool contract exists yet.
- Decision 80's adverse-media/news monitoring — stays fully on hold until CBUAE legal/compliance actually answers whether FPSD may build it.
- Supersession confidence threshold (decision 30) and HITL SLA durations (decision 54) — both "tune empirically once real cases exist," not final numbers.
- ADR-0022's capacity numbers — explicit placeholders, need real annual examination count/concurrency from the examiner team before any GPU procurement decision.
- On-prem platform choice (VMs vs. K8s/OpenShift) and model choice (Qwen Coder vs. gpt-oss-120b, decision 49) — both still open.
- Decision 99's Further-submission review checkpoint needs its SLA duration set — same "tune empirically" treatment as decision 30/54, no number proposed yet.
- Decision 108's Notice Corpus management screen (upload → auto-OCR/segment → human confirm) is scoped as a direction but has no tool contract yet.
- Decision 112's SSO/IdP choice is an open item for CBUAE IT, same treatment as the secrets_manager product choice and the deployment platform.
- Decision 116's unified in-app metrics view — explicitly deferred to a future version, not scoped further now.

## ADR cross-reference

| ADR | Title | Originating decision(s) |
|---|---|---|
| 0001 | LangGraph for orchestration | 8 |
| 0002 | v1 lean end-to-end scope | 22 |
| 0003 | Augmentation not autonomy | 1 |
| 0004 | Citation-grounding hard rule | 4 |
| 0005 | Notice corpus ingestion in v1 | 29 |
| 0006 | Ingestion adapter interface | 14 |
| 0007 | RBAC and audit trail in v1 | 35, 36 |
| 0008 | Single unified app | 34 |
| 0009 | EDM not cross-checked against documents | 20 |
| 0010 | AG and pre-exit revision loops are routine | 37, 38 |
| 0011 | EDM quantitative analysis is not cross-checking | 42 |
| 0012 | Fraud database single physical store | 43, 71 |
| 0013 | Self-hosted LLM in-tenant | 45 |
| 0014 | Fraud database Oracle | 47 |
| 0015 | Deployment target and identity | 44, 46, 48 |
| 0016 | Observability, evaluation, audit stack | 50–54 |
| 0017 | Guardrail enforcement pattern | 56 |
| 0018 | Failure, retry, idempotency semantics | 58 |
| 0019 | Threat model and inference network isolation | 60 |
| 0020 | Severity rubric as shared versioned service | 61 |
| 0021 | Quantitative analysis computation guardrail | 62 |
| 0022 | Capacity and scale placeholder plan | 63 |
| 0023 | Testing, canary, rollback strategy | 64 |
| 0024 | Data retention, lifecycle, audit access | 69 |
| 0025 | LangGraph state schema versioning | 70 |
| 0026 | No dedicated Verifier agent | 77, 78 |
| 0027 | Notice corpus storage strategy confirmed | 79 |
| 0028 | External intelligence scope split | 80 |
| 0029 | Examiner Dashboard userflow architecture | 81–92, 99, 106, 108, 109, 112 |
| 0030 | Prompt injection accepted as bounded risk | 129 |
| 0031 | English-only v1 language scope | 129 |
| 0032 | Fraud Database stores document content, not pointers | 129 |
