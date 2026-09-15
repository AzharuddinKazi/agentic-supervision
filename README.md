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
- **`docs/architecture/lfi-pipeline-v1.html`** — the v1 architecture diagram (self-contained HTML; open directly in a browser — pan/zoom, theme toggle, and export are built in).
- **`.scratch/lfi-examination-pipeline/design-grill.md`** — the full decision log (36 decisions across 7 rounds) behind `CONTEXT.md` and the ADRs. Useful for "why did we land here" archaeology; `CONTEXT.md`/ADRs are the distilled, current reference.

## Status

v1 scope: a lean, end-to-end pipeline (pre-examination intake and gap analysis, examination-meeting support, and post-examination findings/reporting) for banks only, at bare-minimum depth per phase. See `docs/adr/0002-v1-lean-end-to-end-scope.md` for the full scope rationale and what's explicitly deferred to v2.

No build/implementation work has started. The next step is turning the settled design into an implementation plan against the documented architecture.

## Regenerating the architecture diagram

The diagram is built with the vendored `archify` skill:

```bash
cd .claude/skills/archify
npm install
node bin/archify.mjs validate architecture ../../../docs/architecture/lfi-pipeline-v1.candidate.json --quality showcase --json
node bin/archify.mjs deliver  architecture ../../../docs/architecture/lfi-pipeline-v1.candidate.json ../../../docs/architecture/lfi-pipeline-v1.html --quality showcase --json
```

Edit `docs/architecture/lfi-pipeline-v1.candidate.json` and re-run `deliver` to update the diagram.
