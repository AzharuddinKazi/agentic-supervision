# No grounded citation, no compliance verdict

Every compliance verdict must cite the exact notice and the exact clause sentence verbatim — no paraphrasing presented as a quote, since this is regulatory output. We made this a hard rule rather than a best-effort guideline: if the pipeline can't find a high-confidence exact-match citation, it must output "insufficient grounding, needs human review" instead of producing a verdict at all.

This is worth recording because the obvious alternative — let the model produce its best judgment and flag low-confidence ones for review — is what most LLM pipelines default to, and would be wrong here: an ungrounded verdict that looks plausible is a compliance liability, not just a quality issue. The rule directly shaped the notice-corpus requirement (ADR-0005): citations can't be grounded against a corpus that doesn't exist in a queryable, per-clause-addressable form.
