# EDM gets a first-class quantitative analysis track, without reversing the no-cross-checking decision

## Status

Accepted — 2026-09-15. Clarifies, does not reverse, ADR-0009.

## Context

ADR-0009 established that EDM figures are not cross-checked against document claims for consistency. The current-state process flowchart (`docs/reference-docs/current_state.png`) shows Phase 2's findings analysis splitting into **qualitative analysis** (notice/clause compliance narrative) and **quantitative analysis** (EDM figures — fraud volumes, loss amounts, trends/ratios) as two parallel, first-class tracks that both feed findings.

## Decision

We're clarifying, not reversing, ADR-0009: quantitative analysis means analyzing EDM's numbers on their own terms (e.g., a loss ratio increased 40% quarter-over-quarter, worth a finding in its own right) — it does not mean reconciling a document's claimed figures against EDM's reported figures. That reconciliation is still out of scope per ADR-0009's reasoning (the datasets are largely non-overlapping).

## Consequences

This distinction is worth recording because "EDM feeds quantitative analysis" reads, out of context, like exactly the cross-checking ADR-0009 rejected — a future reader implementing this needs to know which one is intended. The quantitative track was later given its own deterministic computation guardrail (ADR-0021) so the LLM never computes a ratio or trend itself from raw figures.
