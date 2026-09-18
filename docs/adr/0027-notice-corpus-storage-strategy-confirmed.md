# Notice/clause corpus storage stays the existing search+exact-lookup+supersession hybrid — no RAG-as-grounding, no knowledge-graph migration

## Status

Accepted — 2026-09-16 (confirms existing architecture; no change)

## Context

The user asked us to research the best storage strategy for the notices/laws/standards the pipeline cites against — weighing RAG (embedding-based semantic retrieval), a structured/relational table, and a knowledge graph. The architecture already has a de facto hybrid: `notice_corpus.search()` (semantic + exact-match retrieval, for *discovery*), `notice_corpus.get_clause()` (exact verbatim lookup, the actual *grounding* mechanism required by ADR-0004), and `supersession_graph.check()` (single-hop clause-to-clause relationship lookup). The question was whether to keep this, or replace it with one of the three canonical approaches.

## Decision

**Keep the existing hybrid. It's already the architecturally correct shape for this domain, not an accident worth replacing.** Research into production legal/compliance citation systems supports this directly: even commercial legal-AI RAG tools hallucinate on 17–33% of queries, and "hallucination with a citation is worse than plain hallucination" is a documented failure mode — a wrong-but-cited verdict is more dangerous than an admitted gap, which is precisely the failure ADR-0004's `insufficient_grounding` fallback exists to prevent. Semantic search belongs only at the discovery step (narrowing candidates), never as the grounding mechanism itself — which is already exactly how `search()` vs. `get_clause()` are split.

**`supersession_graph` is currently a lookup table wearing a graph-shaped API, not a real graph database — this is a deliberate, reasonable v1 scope choice, not a defect.** Nothing in the current spec requires multi-hop traversal (e.g., "list every clause an amendment affects transitively"). A real graph store would only be justified if that requirement appears — don't build it ahead of that need.

## Consequences

This is strong external validation that ADR-0004's hard rule (exact match or explicit refusal, never best-guess) was the right call, and a reason to protect that split going forward, not loosen it toward pure semantic retrieval.

**One real gap, already flagged, now given an explicit resolution path**: notice effective-dating/versioning (ADR-0005's known v1 limitation — current-version-only, no history). External research treats temporal validity as a hard constraint legal citation systems must model explicitly, not something semantic similarity or a supersession graph captures. **When this is picked up, solve it as a structured/relational problem** — `effective_from`/`effective_to` columns on notices and clauses — not as a graph or RAG extension. Supersession answers "which clause overrides which"; effective-dating answers "what was in force on date X," a temporal-range query that belongs on the clause metadata table. `get_clause()`'s current contract (verbatim text, notice reference, no effective-date field) is a natural, low-cost extension point for this later.

No architecture or tool-contract change was required by this ADR — it's a confirmation, with one noted extension point for future work.
