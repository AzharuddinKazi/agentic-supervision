# No dedicated Verifier agent — code guardrails + HITL checkpoints remain the verification mechanism

## Status

Accepted — 2026-09-16

## Context

The user asked us to look at Google's **DS-STAR** paper (*DS-STAR: Data Science Agent via Iterative Planning and Verification*, arXiv 2509.21825, Google Research, Sept 2025) for ideas on evaluating whether an agent's output in this pipeline is actually "done," including whether we should add a dedicated Verifier agent. DS-STAR's loop is Planner → Coder → Verifier (an LLM judge returning sufficient/insufficient) → Router, capped at 10 rounds, terminating on Verifier approval.

**The pattern doesn't transfer directly, and that's diagnostic, not a dead end.** DS-STAR's Verifier judges a runnable, checkable artifact — code executed against real data, producing an inspectable output. This pipeline's outputs (a compliance verdict, a finding's severity, a transmittal-letter draft) are human regulatory judgments with no equivalent "run it and check" step. A literal port — an LLM Verifier approving output and unblocking autonomous continuation — would conflict directly with ADR-0003's augmentation-not-autonomy stance: human judgment is supposed to stay load-bearing, not something an LLM verifier substitutes for.

## Decision

No 6th agent. What already exists is a better-fitted implementation of the same underlying idea, split by what's actually checkable:

1. For things that *are* deterministically checkable (severity matches the rubric lookup, a citation exists and matches, a quantitative figure traces to a `compute()` call), the guardrail pattern (ADR-0017/0020/0021) already does exactly what DS-STAR's Verifier does — check the executed result against ground truth — as a cheap, deterministic code assertion instead of a second LLM call. This is strictly better here: zero added hallucination risk, zero added latency/cost, and it can't itself be wrong about the check the way an LLM judge could.
2. For things that require genuine judgment (is this finding well-reasoned, is this the right supervision question to ask), the existing HITL checkpoints (`agent-specifications.md`'s checkpoint table) are the verification step, by design — an LLM Verifier "approving" these would either duplicate the human checkpoint immediately downstream, or quietly shift the load-bearing judgment from human to LLM, the opposite of ADR-0003.

## Consequences

**One narrower idea is left open, not adopted**: an LLM draft-completeness *pre-check* — not a judge of correctness, just confirming a draft covers every signed-off finding with no dangling references — immediately before the Findings Author sign-off checkpoint, to reduce wasted Lead/Approver review cycles on obviously-incomplete drafts. This would flag only, never gate or loop autonomously, and the human checkpoint would remain the real gate. Not built for v1; revisit if Findings Author drafts are observed wasting reviewer time on completeness issues specifically (a concrete signal to wait for, not a guess).

`agent-specifications.md`'s "Definition of done, per agent" section — the concrete, checkable completion predicate this decision produced for each agent — is the cheap half of the original ask, now written down explicitly instead of left implicit across ADR-0017/0018/0020/0021 and the shared-state table.
