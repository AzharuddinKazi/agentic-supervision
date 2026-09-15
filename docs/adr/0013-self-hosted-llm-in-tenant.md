# LLM is self-hosted, in-tenant, not a third-party API

Given LFI examination data is sensitive supervisory data (decision 2 — must run in an approved/sandboxed enterprise environment), the LLM powering every agent (ADR-0001's LangGraph nodes) is a self-hosted open-weight model served on infrastructure inside CBUAE's own tenant, not a call to a third-party model API. No examination data leaves the enterprise boundary at inference time, including for the citation-grounding (ADR-0004) and quantitative-analysis (ADR-0011) steps.

This is worth recording because it shapes real infrastructure: a GPU-backed inference service is now a first-class deployment component (see the deployment architecture), not an external dependency to just call and forget.

**Which model** stays a deliberate black box behind that inference service — the pipeline's agents (`agent-specifications.md`) are written against a model-agnostic interface, not a specific model's quirks. IT already has two candidates available on-prem: **Qwen Coder** and **gpt-oss-120b**. Neither is committed to yet; model-quality tooling (evals, prompt iteration) should be built to swap between candidates rather than assume one, since this is explicitly a "revisit later" decision.
