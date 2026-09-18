# v1 targets English-language content only — an explicit decision, not an unexamined assumption

## Status

Accepted, pending FPSD confirmation — 2026-09-17

## Context

An independent architecture review (Principal Engineer audit, `docs/reviews/2026-09-17-principal-engineer-audit.md`, High Finding 5.2) found zero occurrences of Arabic, bilingual, localization, RTL, or language-handling anywhere across `CONTEXT.md`, all 29 ADRs, `agent-specifications.md`, `tool-contracts.md`, and the decision log — for a UAE central bank regulator, where this is not a polish item but an architecture input in at least four places.

## Decision

**v1 is scoped to English-language notices, RFI documents, and meeting minutes only.** This is stated explicitly here, as a decision to confirm with FPSD before Notice Corpus ingestion begins in earnest — the same "ask CBUAE, don't guess a product into the diagram" treatment already given to the SSO/IdP choice (ADR-0015), the on-prem platform choice (ADR-0015), and the secrets-manager product (ADR-0019). It is not being silently assumed correct.

**Why this needed a decision rather than staying implicit, and where it bites if wrong:**
1. **OCR (ADR-0005/0006).** Arabic OCR quality and tooling are materially different from English OCR — a scanned Arabic notice run through an English-tuned OCR pipeline degrades silently, producing plausible-looking but wrong segmented clauses.
2. **Citation-grounding guardrail (ADR-0004).** The grounding rule requires a **verbatim exact match** of a clause sentence — a code-level equality/near-equality check. Exact-match behavior across scripts, diacritics, and Unicode normalization forms (e.g. NFC vs. NFD) is a real, separate engineering problem for Arabic text that the current guardrail implementation does not address.
3. **Model selection (decision 49, ADR-0013).** Qwen Coder vs. gpt-oss-120b is currently framed as a swappable, model-agnostic choice. If Arabic-language competence enters scope, it becomes a first-order selection criterion, not a detail the model-agnostic interface can sidestep.
4. **Transmittal letter / pre-exit deck templates.** Fixed Word/PPT templates (`template_fill`, `tool-contracts.md`) are implicitly English-layout; a bilingual or Arabic-primary template is a distinct template asset, not a translation of the existing one.

## Consequences

**Revisit trigger**: before Notice Corpus ingestion begins in earnest (the point at which real notice documents start flowing through OCR), get explicit confirmation from FPSD on whether any in-scope notices, RFI documents, or examiner-facing output are Arabic or bilingual. If yes, items 1–4 above each need their own follow-up decision before the affected component ships — this ADR does not attempt to answer them preemptively, since the honest answer today is "unconfirmed," not "no."
