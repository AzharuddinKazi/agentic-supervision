# Tool contracts

Every tool named in `agent-specifications.md` gets a real contract here: input schema, output schema (including a distinct error/timeout shape from "no result found"), timeout, and retry policy. This is the fix for the independent review's High Finding #6 — "tool contracts are prose, not contracts." Read `docs/adr/0018-failure-retry-idempotency-semantics.md` first for the cross-cutting policies these contracts all implement.

**The one rule that applies to every tool below:** a timeout or connection error is a distinct outcome from "no matching result," represented by a distinct `status` value (`"error"`, never folded into `"ok"` with an empty payload). This matters most for `notice_corpus.get_clause` under the citation-grounding rule (ADR-0004) and `edm.query` under the quantitative-analysis guardrail (ADR-0021): treating a timeout as "no data exists" produces a wrong, confidently-stated verdict instead of a correct one that was merely delayed and should be retried.

---

## `log_checkpoint(state_before, state_after, actor, event_type, resolution?, edit_diff?, reason_code?)`
**Used by**: every agent, at every `interrupt()` resolution and every state-machine transition (`agent-specifications.md`'s Audit Trail Store section). The single write path into `audit_log[]` — no agent writes an `AuditLogEntry` any other way (fixes Principal Engineer audit 3.2's finding that `lfi-guardrail-citation-grounding-v1` drew a direct `gap_analysis → audit_trail` message instead of routing through this call; that diagram is corrected to route through `log_checkpoint()` like everywhere else).
**Input**: `{ state_before: object, state_after: object, actor: { type: "agent"|"human", name: string, user_id?: string }, event_type: string, resolution?: "accept"|"edit"|"reject", edit_diff?: object, reason_code?: string }` — field meanings match the `AuditLogEntry` schema in `agent-specifications.md`. `idempotency_key`, `prev_hash`, and `entry_hash` are **not** caller-supplied; this call derives them internally from the current LangGraph node-execution context and the examination's existing chain tail.
**Output (ok)**: `{ status: "ok", entry_id: string, entry_hash: string }`.
**Output (error)**: `{ status: "error", reason: "duplicate_idempotency_key" | "chain_integrity_violation" }` — `duplicate_idempotency_key` is a benign no-op (the entry already exists from a prior attempt at this exact node execution; the caller should treat it as success and use the returned existing `entry_id`, not retry). `chain_integrity_violation` (the computed `prev_hash` doesn't match the chain's current tail) is a correctness alarm, not a transient failure — it means either concurrent unserialized writes to the same examination's chain or actual tampering, and must halt and page, never silently retry past it.
**Timeout**: 2s (a DB insert, not an LLM call).
**Retry**: 3 attempts, backoff (0.5s/1s/2s), transient DB errors only — never retried on `chain_integrity_violation`.

---

## `ingestion_adapter.list_documents(lfi_id)`
**Used by**: Intake Tracker.
**Input**: `{ lfi_id: string }`
**Output (ok)**: `{ status: "ok", documents: [{ doc_id: string, filename: string, path: string }] }`
**Output (error)**: `{ status: "error", reason: "adapter_unreachable" | "auth_failed" | "invalid_lfi_id" }`
**Timeout**: 10s (SFTP directory listing).
**Retry**: 3 attempts, exponential backoff (1s/2s/4s), transient only (`adapter_unreachable`); `auth_failed`/`invalid_lfi_id` are terminal, no retry — escalate to human review of the ingestion configuration.

## `ingestion_adapter.get_metadata(doc_id)`
**Used by**: Intake Tracker.
**Input**: `{ doc_id: string }`
**Output (ok)**: `{ status: "ok", size_bytes: int, filename: string, last_modified: ISO8601 }`
**Output (error)**: `{ status: "error", reason: "adapter_unreachable" | "not_found" }`
**Timeout**: 5s.
**Retry**: 3 attempts, same backoff as above, transient only. `not_found` is terminal — the document was listed but vanished before metadata fetch; treat as `pending`, not `suspicious` (a race with the LFI's own upload process, not evidence of a malformed submission).

## `notice_corpus.search(query, clause_filter)`
**Used by**: Compliance Analyst.
**Input**: `{ query: string, clause_filter?: string }`
**Output (ok)**: `{ status: "ok", results: [{ clause_id: string, score: float, snippet: string }] }` (may be an empty `results[]` — that is a legitimate "no match" result, distinct from `status: "error"`)
**Output (error)**: `{ status: "error", reason: "corpus_unavailable" | "index_rebuilding" }`
**Timeout**: 8s.
**Retry**: 4 attempts, exponential backoff (1s/2s/4s/8s), transient only. `index_rebuilding` retries with a longer backoff (30s) since it's a known, bounded-duration state, not a failure.

## `notice_corpus.get_clause(clause_id)`
**Used by**: Compliance Analyst — **the citation-grounding rule's tool** (ADR-0004).
**Input**: `{ clause_id: string }`
**Output (ok, match)**: `{ status: "ok", found: true, clause_id: string, verbatim_text: string, notice_ref: string }`
**Output (ok, no match)**: `{ status: "ok", found: false }` — this is what actually triggers `insufficient_grounding` (ADR-0004). **A real absence, established affirmatively by the corpus, not inferred from a failed call.**
**Output (error)**: `{ status: "error", reason: "corpus_unavailable" | "index_rebuilding" }` — **must never be treated as `found: false`.** Compliance Analyst's node must branch on `status` before it ever looks at `found`.
**Timeout**: 5s.
**Retry**: 4 attempts, exponential backoff (1s/2s/4s/8s). If all retries exhaust: the clause-verdict loop iteration for that RFI question is deferred (re-queued, not failed-closed to `insufficient_grounding`) and surfaced on the Examiner Dashboard as a corpus-availability issue distinct from a grounding gap — logged separately in `audit_log[]` per `agent-specifications.md`'s merged "Human Review Flags" note.

## `edm.query(lfi_id, period)`
**Used by**: Compliance Analyst (context), Findings Author (quantitative track input).
**Input**: `{ lfi_id: string, period: { from: ISO8601, to: ISO8601 } }`
**Output (ok)**: `{ status: "ok", figures: { business_volume: number, fraud_transactions_cards: number, fraud_transactions_transfers: number, ... } }`
**Output (error)**: `{ status: "error", reason: "edm_unreachable" | "no_data_for_period" }` — `no_data_for_period` is a legitimate business outcome (LFI hasn't reported yet for that quarter), distinct from `edm_unreachable`, which is transient infrastructure failure. **The two must never share a code path** — the same principle as `notice_corpus.get_clause`, applied to the quantitative track's data source.
**Timeout**: 10s (Oracle API round-trip).
**Retry**: 3 attempts, exponential backoff (2s/4s/8s), `edm_unreachable` only. `no_data_for_period` is terminal — the quantitative track for that period is skipped, not retried, and this must be visible in the finding rather than silently omitted.

## `supersession_graph.check(clause_id)`
**Used by**: Compliance Analyst.
**Input**: `{ clause_id: string }`
**Output (ok)**: `{ status: "ok", superseded: bool, superseding_clause_id?: string, confidence: float }` — `confidence` below the configured threshold (decision 30) routes to `needs_human_review` per `agent-specifications.md`, not resolved silently.
**Output (error)**: `{ status: "error", reason: "graph_unavailable" }`
**Timeout**: 5s.
**Retry**: 3 attempts, exponential backoff (1s/2s/4s). Exhausted retries route to `needs_human_review` with reason `graph_unavailable` — same "don't silently guess" principle, applied to supersession instead of raw citation lookup.

## `severity_rubric.lookup(finding, license_type, rubric_version?)`
**Used by**: Findings Author. **Shared, versioned platform service** (ADR-0020) — not owned by any one agent. Scoped per `license_type` (**resolves Principal Engineer audit 4.6/6.5** — the rubric is one shared service but a distinct versioned rubric per license type, per `agent-specifications.md`'s Settings/Admin section; without this parameter a second license type could never be looked up correctly through this contract).
**Input**: `{ finding: FindingSummary, license_type: string, rubric_version?: string }` — omitting `rubric_version` uses that license type's current published version.
**Output (ok)**: `{ status: "ok", severity: "low"|"medium"|"high", deadline_days: int, rubric_version: string }` — the returned `rubric_version` is what must be persisted alongside `severity` (ADR-0020's guardrail). Three values, lowercase wire form of `CONTEXT.md`'s canonical High/Medium/Low — no fourth "critical" tier (fixes Principal Engineer audit 4.1, which found this contract's four-value enum silently contradicting `CONTEXT.md`'s three-value definition; a code-enforced equality guardrail on this field means the mismatch fails loudly and late otherwise).
**Output (error)**: `{ status: "error", reason: "rubric_service_unavailable" | "no_matching_rule" }` — `no_matching_rule` means the rubric has a genuine gap for this finding type; this is a data/config issue, not a transient failure, and routes to human review (a Lead/Approver assigns severity manually and this is flagged as a rubric-coverage gap to fix).
**Timeout**: 3s (low-latency lookup service, not an LLM call).
**Retry**: 3 attempts, backoff (0.5s/1s/2s), `rubric_service_unavailable` only.

## `quant_analysis.compute(lfi_id, period_range, metric)`
**Used by**: Findings Author. **Deterministic — code, not LLM** (ADR-0021).
**Input**: `{ lfi_id: string, period_range: { from: ISO8601, to: ISO8601 }, metric: "loss_ratio"|"volume_trend"|"fraud_rate_delta" }`
**Output (ok)**: `{ status: "ok", value: number, unit: string, basis: { numerator_query: EdmQueryRef, denominator_query: EdmQueryRef } }` — `basis` makes the computation traceable back to the exact `edm.query` calls it was derived from, mirroring the citation-traceability principle ADR-0004 established for the qualitative track.
**Output (error)**: `{ status: "error", reason: "insufficient_data" | "edm_unreachable" }` — `insufficient_data` (e.g., can't compute a quarter-over-quarter delta with only one quarter of data) is terminal for that metric; `edm_unreachable` is transient.
**Timeout**: 12s (may issue multiple underlying `edm.query` calls).
**Retry**: 3 attempts, backoff (2s/4s/8s), `edm_unreachable` only.
**Guardrail**: no quantitative finding may be persisted with a number not traceable to a `value` returned by this tool (ADR-0021) — enforced in code at the same write-time check as the severity/rubric-version guardrail (ADR-0017, ADR-0020).

## `template_fill(template_id, findings[])`
**Used by**: Findings Author.
**Input**: `{ template_id: "transmittal_letter"|"pre_exit_deck", findings: FindingRecord[] }`
**Output (ok)**: `{ status: "ok", document_ref: string }` — a reference (not the raw bytes) into wherever drafted documents are stored.
**Output (error)**: `{ status: "error", reason: "template_service_unavailable" | "template_not_found" | "schema_mismatch" }` — `schema_mismatch` means a `FindingRecord` is missing a field the fixed Word/PPT template requires; this is a structured-output validation failure (ADR-0018 policy 4), not a template-service problem, and should trigger the one-retry repair loop before falling back to human escalation.
**Timeout**: 15s (document generation, not just a lookup).
**Retry**: 2 attempts, backoff (2s/4s), `template_service_unavailable` only. `template_not_found`/`schema_mismatch` are terminal — configuration/data problems, not transient failures.

---

## Cost consideration (LLM Inference Service calls specifically)

Every LLM call underlying Compliance Analyst's per-clause ReAct loop, Meeting Facilitator's drafting/extraction, Findings Author's narration, and Approval Coordinator's feedback summarization has a GPU-time cost that the retry policies above must respect: **retries above are for the deterministic tools these agents call, not for the LLM call itself.** An LLM call that returns malformed structured output gets exactly one repair retry (ADR-0018 policy 4) — not the same bounded-backoff retry loop as a tool timeout, since a malformed *output* is a different failure mode from an *unreachable* service, and unbounded LLM-call retries on a per-clause loop running up to ~40 times per examination (ADR-0022's capacity estimate) would multiply GPU cost silently. A second malformed-output failure escalates to human review; it does not retry again.
