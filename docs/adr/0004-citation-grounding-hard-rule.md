# No grounded citation, no compliance verdict

## Status

Accepted — 2026-09-15

## Context

Every compliance verdict must cite the exact notice and the exact clause sentence verbatim — no paraphrasing presented as a quote, since this is regulatory output. The obvious alternative — let the model produce its best judgment and flag low-confidence ones for review — is what most LLM pipelines default to, and would be wrong here: an ungrounded verdict that looks plausible is a compliance liability, not just a quality issue.

## Decision

We made this a hard rule rather than a best-effort guideline: if the pipeline can't find a high-confidence exact-match citation, it must output "insufficient grounding, needs human review" instead of producing a verdict at all.

## Consequences

The rule directly shaped the notice-corpus requirement (ADR-0005): citations can't be grounded against a corpus that doesn't exist in a queryable, per-clause-addressable form. It was later generalized into a standing pattern for every agent (ADR-0017), and external research subsequently validated the underlying "exact match or refuse" approach over semantic/RAG-based grounding (ADR-0027).
