# Quantitative findings require a deterministic computation tool — LLMs narrate numbers, never compute them

## Status

Accepted — 2026-09-15

## Context

An independent architecture review (`.scratch/architecture-review-2026-09-15.md`, High Finding #8) found that Findings Author's quantitative track — "EDM figures analyzed on their own terms (loss ratios, volume trends)," ADR-0011's own example being "a loss ratio increased 40% quarter-over-quarter" — had no tool for doing the actual arithmetic. Only `edm.query` (a read-only fetch) and `severity_rubric.lookup`/`template_fill` existed. That leaves it ambiguous whether a specific percentage presented as fact in a regulatory finding is computed deterministically in code, or computed by the LLM directly from raw figures in its context window — the latter being a known, well-documented source of silent arithmetic errors, and exactly the shape of risk ADR-0004's citation-grounding rule already exists to prevent for the *qualitative* track. The quantitative track had no equivalent protection.

## Decision

Add `quant_analysis.compute(lfi_id, period_range, metric)` as a deterministic (code, not LLM) tool on Findings Author, returning pre-computed ratios/trends/deltas from EDM figures. The LLM's role in the quantitative track is narrowed to **narrating and prioritizing** these pre-computed numbers into findings prose — it never computes a ratio, delta, or trend itself from raw figures in-context.

**New guardrail, parallel to ADR-0004/ADR-0017's pattern**: no quantitative finding may be persisted with a number that is not traceable to a specific `quant_analysis.compute()` call return value — enforced the same way severity is checked against `severity_rubric.lookup()` (ADR-0017, ADR-0020): in code, before the write, not left to the model's narration being trusted at face value.

## Consequences

A hallucinated or miscalculated percentage in a regulatory finding is indistinguishable, on the page, from a correct one — the transmittal letter and pre-exit deck present it as fact to LFI leadership and the Assistant Governor. This is the same failure mode ADR-0004 named for citations ("an ungrounded verdict that looks plausible is a compliance liability, not just a quality issue"), just on the numeric side of the findings split (ADR-0011) instead of the qualitative side. `quant_analysis.compute()`'s output schema, timeout, and retry policy are specified in `docs/architecture/tool-contracts.md` alongside every other tool (High Finding #6).
