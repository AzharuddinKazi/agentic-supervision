# Prompt injection via LFI-authored content is an accepted, bounded risk for v1 — not an unexamined gap

## Status

Accepted — 2026-09-17

## Context

An independent architecture review (Principal Engineer audit, `docs/reviews/2026-09-17-principal-engineer-audit.md`, High Finding 5.1) found zero mention of prompt injection anywhere across 29 ADRs, `agent-specifications.md`, and `tool-contracts.md` — despite the pipeline feeding **adversary-authored content** into LLM agents: documents uploaded by the bank under examination (which has a direct material interest in the verdict), free-text meeting minutes, and free-text AG feedback. ADR-0019's threat-model deferral covers malicious *file upload* (malware, parser exploits) — a different threat, not this one.

## Decision

**Accepted risk for v1, bounded by existing guardrails, with an explicit revisit trigger** — not silently absent, and not a blocker to starting development.

**Why the existing guardrails materially bound the blast radius:**
- ADR-0004's citation-grounding rule means an injected instruction cannot fabricate a citation that doesn't exist in Notice Corpus — the worst a manipulated document can do on the citation axis is push toward `insufficient_grounding`, which routes to human review, not toward a fabricated-but-confident verdict.
- Every LLM output (compliance verdicts, supervision questions, findings, AG-feedback summaries) passes at least one HITL checkpoint (`agent-specifications.md`'s HITL table) before it becomes binding — there is no path from "LLM read adversarial input" to "regulatory action" without a human in between.

## Consequences

**What this does not cover, and is explicitly not claiming to:**
- Severity inflation or deflation via injected instructions that stay within otherwise-plausible language (a human reviewer checking severity against a rubric may not catch subtle steering).
- Supervision questions being steered away from a real gap the LFI would rather not have probed.
- A "compliant" verdict on a clause that genuinely exists in the corpus, where the injected content manipulates *interpretation* rather than fabricating a citation.

**Revisit trigger**: fold an LLM prompt-injection test suite — adversarial inputs shaped like real LFI documents and meeting-minute text — into the pre-go-live security exercise ADR-0019 already schedules (STRIDE pass, ingestion-path analysis). Also revisit if the human-agreement-rate metric (ADR-0016) ever shows an anomaly correlated with a specific LFI's document set, since that's the closest early-warning signal this risk would actually produce. No code change is implied by this ADR; it exists so the risk is named and owned, not missing.
