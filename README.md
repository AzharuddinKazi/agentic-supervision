# Agentic Supervision

Design documentation for a multi-agent pipeline that augments FPSD (Fraud Prevention and Supervision Department) at CBUAE in running LFI (Licensed Financial Institution) compliance examinations.

This repo is currently **documentation and design artifacts only** — no application code has been built yet. It captures the outcome of a structured design-tree interview ("design grill") and the resulting domain model, architectural decisions, and a v1 architecture diagram.

## Repo layout

```
CONTEXT.md                     Domain glossary — canonical vocabulary for this project
docs/
  decision-log.md               Every question asked and decision made, in order, cross-referenced to ADRs
  adr/                          Architecture Decision Records (numbered, sequential)
  architecture/                 Generated architecture diagrams (source spec + delivered HTML)
  agents/                       Conventions for how AI coding agents should use this repo's docs
.scratch/
  lfi-examination-pipeline/     Raw working decision log (gitignored) — docs/decision-log.md is its committed, canonical copy
.claude/skills/archify/         Vendored third-party skill used to generate the architecture diagrams
.agents/skills/                 Vendored skill definitions used during doc co-authoring
CLAUDE.md                       Project instructions for AI coding agents working in this repo
```

## Where to start

- **`CONTEXT.md`** — read this first. It defines every domain term (LFI, RFI, notice/clause, supersession, EDM, finding, transmittal letter, etc.) as used in this project.
- **`docs/decision-log.md`** — every question raised and decision made, in order (80 decisions across 17 rounds as of 2026-09-16), cross-referenced to ADRs. This is the committed, durable record — check it before reopening anything that looks settled, so a prior decision doesn't get silently re-litigated.
- **`docs/adr/`** — read the ADRs relevant to the area you're touching before making architectural changes. Each one records a decision, its rationale, and what alternative was rejected and why. `docs/decision-log.md` has a full ADR cross-reference table at the bottom.
- **`docs/architecture/lfi-pipeline-context-v1.html`** — the v1 C4-style Context diagram: the pipeline as one system plus every external actor/system it touches (LFI, Shared Workspace, Notice source, EDM, SSO, FPSD examiner team, Assistant Governor). Start here for the fully-zoomed-out view before dropping into the container-level diagram below.
- **`docs/architecture/lfi-pipeline-v1.html`** — the v1 logical architecture diagram, phase-segregated to match `current_state.png`'s own phase names (Sending out the RFIs / Examination Phase Onsite / Pre-Exit Phase). Color encodes automation role, not just architecture layer — see the diagram's own legend and "How to read the colors" card: grey = human step or external system, teal = LLM agent (reasoning only), peach = LLM agent + tool call, gold = deterministic tool/rule-based service, violet = data store, blue = human-facing UI.
- **`docs/architecture/lfi-pipeline-deployment-v1.html`** — the v1 deployment & ownership view (4 LLM agents + 1 rule-based service individually shown with persona/tools, infra ownership, network zones, observability/eval stack). Named "deployment" but not a strict UML deployment diagram — no per-service execution-node/replica modeling, since the actual deployment target is still open (ADR-0015); revisit once that's chosen. Shares the logical diagram's automation-role color legend, extended with a `security` category for platform/infra services that aren't agents.
- **`docs/architecture/lfi-workflow-phase{1,2,3}-*-v1.html`** — the actual LangGraph node/edge/conditional-transition diagrams, one per phase (component diagrams show *what exists*; these show *how it actually runs*, including every guardrail branch, human checkpoint, and the Phase 2 sufficiency-check gate).
- **`docs/architecture/lfi-guardrail-citation-grounding-v1.html`** — a UML-style sequence diagram for the single highest-risk interaction in the spec: Compliance Analyst's citation-grounding call flow, including the retry-vs-real-absence distinction (ADR-0004/ADR-0018) that a tool timeout must never collapse into.
- **`docs/architecture/agent-specifications.md`** — per-agent contract: trigger, inputs, outputs, tools, model behavior rules (including the failure/retry/idempotency policy, ADR-0018), shared state, human checkpoints with SLAs, capacity/testing/threat-model summaries. Read this before implementing any agent.
- **`docs/architecture/tool-contracts.md`** — one-page contract per tool (input/output schema, timeout, retry policy, and the "unavailable ≠ no result" distinction) — read this before writing any tool-calling code.
- **`.scratch/architecture-review-2026-09-15.md`** — an independent staff-engineer-level production-readiness review (gitignored, referenced in `docs/decision-log.md` Rounds 12/13/15). All 4 Critical, all 6 High, and all 5 Medium findings are fixed; only 2 Low findings remain open (cost model, independent model-risk/compliance sign-off process), both organizational rather than architectural. **Note for future diagram work**: archify has no ER/crow's-foot-capable diagram type (`architecture | workflow | sequence | dataflow | lifecycle`) — when the Fraud Database's Oracle schema is eventually diagrammed, it needs a different tool (e.g. Mermaid `erDiagram`), not archify's `dataflow` type forced to look like one.

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
