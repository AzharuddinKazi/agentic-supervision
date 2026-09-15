# Capacity and scale plan (placeholder, pending real usage data)

An independent architecture review (`.scratch/architecture-review-2026-09-15.md`, High Finding #9) found no concurrent-examination target, no GPU sizing methodology, and no Oracle storage/IOPS estimate anywhere — "self-hosted, GPU" (ADR-0013) and "Oracle" (ADR-0014) are engine choices, not a capacity plan, and CBUAE needs to actually provision hardware before go-live. The review's own recommendation is to produce a rough order-of-magnitude estimate now, explicitly marked for revisit once real usage data exists — the same treatment decision 30 already gives the supersession-confidence threshold ("tune empirically; start conservative").

**We do not have FPSD's real annual examination count or concurrency from the examiner team yet** — this ADR states the sizing *methodology* and a placeholder number, not a validated capacity plan. Get the real inputs (below) from the examiner team before procurement, not from this document.

**Methodology:**
```
GPU count  ≈  (peak concurrent Gap Analysis ReAct loops)
              × (avg tokens per clause-verdict call)
              × (target p95 latency budget)
              ÷ (single-GPU sustained throughput at that model size)

Oracle storage/IOPS  ≈  (annual examination count)
              × (avg documents + findings + audit_log entries per examination)
              × (avg row/blob size)
              , with IOPS driven primarily by audit_log[] append rate at peak concurrency
```

**Placeholder inputs (illustrative only, not sourced from FPSD's real workload — replace before procurement):**
- Peak concurrent examinations: **8** (rough guess — a mid-size supervisory department running staggered 30-day RFI windows across a bank population).
- Gap Analysis Agent's ReAct loop runs once per RFI question/clause pair (`agent-specifications.md`); assume ~40 RFI questions per examination → up to ~320 concurrent per-clause LLM calls at peak, though these queue rather than all firing simultaneously.
- Avg tokens per clause-verdict call: ~2,000 (prompt + completion) — this depends heavily on which model (Qwen Coder vs. gpt-oss-120b, decision 49) is actually selected; both are named as candidates but neither is committed to, so this number moves once that's decided.
- Target p95 latency: a few seconds per clause-verdict call is likely acceptable given the human review step immediately downstream — no hard SLA has been set by the examiner team yet.
- Resulting placeholder: **2-4 GPUs** for the LLM Inference Service at this scale, **revisit once real concurrency and model choice are known.**
- Oracle: assume low hundreds of MB per examination (documents are referenced/metadata-tracked, not necessarily stored as blobs in Oracle itself — depends on whether the Fraud Database stores document content or just pointers into the Shared Workspace/Smart Portal, which is **not yet decided** and should be resolved before this number is trusted) — placeholder low-tens-of-GB/year, unlikely to be the capacity bottleneck compared to GPU sizing.

**Observability hook-up:** the GPU "queue depth" metric already specified in `agent-specifications.md`'s observability section needs an actual alert threshold once real sizing is known — currently unset because there's no baseline to set it against.

**Why this matters:** without even an order-of-magnitude number, GPU procurement can't start, and a late "we need N more GPUs" discovery after go-live is a much more expensive place to find this out than during architecture review.

**Next step, not done here:** ask FPSD's examiner team for (a) actual annual bank-examination count, (b) typical concurrency (how many examinations are genuinely in-flight at once, not just scheduled across the year), and (c) any existing latency expectation for examiner-facing turnaround. Replace every placeholder number above once those answers exist, and note the replacement in this ADR rather than a silent edit, the same way decision 30's threshold is expected to be tuned empirically and recorded.
