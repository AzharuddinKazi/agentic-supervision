# Agentic Supervision

Design documentation for a multi-agent pipeline that augments FPSD (Fraud Prevention and Supervision Department) at CBUAE in running LFI (Licensed Financial Institution) compliance examinations.

This repo is currently **documentation and design artifacts only** — no application code has been built yet. It captures the outcome of a structured design-tree interview ("design grill") and the resulting domain model, architectural decisions, and a v1 architecture diagram.

## Repo layout

```
CONTEXT.md                     Domain glossary — canonical vocabulary for this project
docs/
  adr/                         Architecture Decision Records (numbered, sequential)
  architecture/                Generated architecture diagram (source spec + delivered HTML)
  agents/                      Conventions for how AI coding agents should use this repo's docs
.scratch/
  lfi-examination-pipeline/    Full working decision log from the design-grill session
.claude/skills/archify/        Vendored third-party skill used to generate the architecture diagram
.agents/skills/                Vendored skill definitions used during doc co-authoring
CLAUDE.md                      Project instructions for AI coding agents working in this repo
```

## Where to start

- **`CONTEXT.md`** — read this first. It defines every domain term (LFI, RFI, notice/clause, supersession, EDM, finding, transmittal letter, etc.) as used in this project.
- **`docs/adr/`** — read the ADRs relevant to the area you're touching before making architectural changes. Each one records a decision, its rationale, and what alternative was rejected and why.
- **`docs/architecture/lfi-pipeline-context-v1.html`** — the v1 C4-style Context diagram: the pipeline as one system plus every external actor/system it touches (LFI, Shared Workspace, Notice source, EDM, SSO, FPSD examiner team, Assistant Governor). Start here for the fully-zoomed-out view before dropping into the container-level diagram below.
- **`docs/architecture/lfi-pipeline-v1.html`** — the v1 logical architecture diagram, phase-segregated to match `current_state.png`'s own phase names (Sending out the RFIs / Examination Phase Onsite / Pre-Exit Phase). Color encodes automation role, not just architecture layer — see the diagram's own legend and "How to read the colors" card: grey = human step or external system, teal = LLM agent (reasoning only), peach = LLM agent + tool call, gold = deterministic tool/rule-based service, violet = data store, blue = human-facing UI.
- **`docs/architecture/lfi-pipeline-deployment-v1.html`** — the v1 deployment & ownership view (4 LLM agents + 1 rule-based service individually shown with persona/tools, infra ownership, network zones, observability/eval stack). Named "deployment" but not a strict UML deployment diagram — no per-service execution-node/replica modeling, since the actual deployment target is still open (ADR-0015); revisit once that's chosen. Shares the logical diagram's automation-role color legend, extended with a `security` category for platform/infra services that aren't agents.
- **`docs/architecture/lfi-workflow-phase{1,2,3}-*-v1.html`** — the actual LangGraph node/edge/conditional-transition diagrams, one per phase (component diagrams show *what exists*; these show *how it actually runs*, including every guardrail branch, human checkpoint, and the Phase 2 sufficiency-check gate).
- **`docs/architecture/lfi-guardrail-citation-grounding-v1.html`** — a UML-style sequence diagram for the single highest-risk interaction in the spec: Compliance Analyst's citation-grounding call flow, including the retry-vs-real-absence distinction (ADR-0004/ADR-0018) that a tool timeout must never collapse into.
- **`docs/architecture/agent-specifications.md`** — per-agent contract: trigger, inputs, outputs, tools, model behavior rules (including the failure/retry/idempotency policy, ADR-0018), shared state, human checkpoints with SLAs, capacity/testing/threat-model summaries. Read this before implementing any agent.
- **`docs/architecture/tool-contracts.md`** — one-page contract per tool (input/output schema, timeout, retry policy, and the "unavailable ≠ no result" distinction) — read this before writing any tool-calling code.
- **`.scratch/architecture-review-2026-09-15.md`** — an independent staff-engineer-level production-readiness review. All 4 Critical, all 6 High, and all 5 Medium findings are now fixed (ADR-0017 through ADR-0025, the three workflow diagrams, `tool-contracts.md`) — only the review's 2 Low findings remain open (cost model, independent model-risk/compliance sign-off process), both organizational rather than architectural.
- A second independent review (a solution architect, fresh agent, 2026-09-16) audited the diagrams for notation/standard conformance — see each diagram's own "How to read this diagram" card for the tool-driven deviations it found (no decision-diamond primitive in the workflow diagrams; the deployment diagram's scope caveat above). One real cross-diagram bug it caught (the deployment diagram's colors didn't carry the logical diagram's automation-role legend, misleadingly coloring `Intake Tracker` the same as the real LLM agents) is fixed, and the two cheap missing artifacts it flagged (a C4 Context diagram, a sequence diagram for the citation-grounding guardrail) are now both added above. **Note for future diagram work**: archify has no ER/crow's-foot-capable diagram type (`architecture | workflow | sequence | dataflow | lifecycle`) — when the Fraud Database's Oracle schema is eventually diagrammed, it needs a different tool (e.g. Mermaid `erDiagram`), not archify's `dataflow` type forced to look like one.
- **ADR-0026 through ADR-0028** (2026-09-16) — three more decisions from user-directed research: no dedicated Verifier agent (existing code guardrails + HITL checkpoints already do that job; DS-STAR's pattern doesn't port cleanly to a human-augmentation system) plus an explicit per-agent "definition of done" in `agent-specifications.md`; the notice/clause corpus's existing search+exact-lookup+supersession hybrid confirmed correct (no RAG-as-grounding, no knowledge-graph migration); external-intelligence enrichment (fraud news, sanctions, consumer complaints) split into sanctions/watchlist screening (scoped as a near-term addition) vs. adverse-media/news monitoring (deferred pending CBUAE legal/compliance sign-off).
- **`.scratch/lfi-examination-pipeline/design-grill.md`** — the full decision log (79+ decisions across 17 rounds) behind `CONTEXT.md` and the ADRs — this **is** the running question/answer record so prior decisions aren't silently re-litigated; read it before reopening something that looks settled. `CONTEXT.md`/ADRs are the distilled, durable reference; the design-grill log is the fuller "why did we land here, what was considered and rejected" archive behind them.

## Status

v1 scope: a lean, end-to-end pipeline (pre-examination intake and gap analysis, examination-meeting support, and post-examination findings/reporting) for banks only, at bare-minimum depth per phase. See `docs/adr/0002-v1-lean-end-to-end-scope.md` for the full scope rationale and what's explicitly deferred to v2. The deployment target (specific cloud/on-prem platform) is intentionally left open — see `docs/adr/0015-deployment-target-and-identity.md`.

No build/implementation work has started. The next step is turning the settled design into an implementation plan against the documented architecture and agent specifications.

## Regenerating a diagram

All diagrams are built with the vendored `archify` skill and named `<subject>-v1.<ext>` throughout. Architecture diagrams (`lfi-pipeline-v1`, `lfi-pipeline-deployment-v1`, `lfi-pipeline-context-v1`) use `diagram_type: "architecture"`; the LangGraph diagrams (`lfi-workflow-phase*-v1`) use `diagram_type: "workflow"`; `lfi-guardrail-citation-grounding-v1` uses `diagram_type: "sequence"`:

```bash
cd .claude/skills/archify
npm install
node bin/archify.mjs validate <architecture|workflow> <path-to-candidate.json> --quality showcase --json
node bin/archify.mjs deliver   <architecture|workflow> <path-to-candidate.json> <path-to-output.html> --quality showcase --json
```

Edit the relevant `docs/architecture/*.candidate.json` and re-run `deliver` to update the diagram. The deployment diagram additionally sets `meta.engineering_profile: "deployment-ownership"`, which enforces real deployment hygiene at validation time: every non-external component needs an owning team (`tag`) and exactly one region assignment, and every stateful component needs a private security-group boundary.
