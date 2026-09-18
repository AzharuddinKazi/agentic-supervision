# External intelligence enrichment splits into two proposals with different scope — sanctions screening is v1-adjacent, adverse-media monitoring is deferred pending legal sign-off

## Status

Accepted — 2026-09-16

## Context

The user asked about adding a web-research/scraping capability to surface external signals about the LFI under examination — fraud-related news, consumer complaints, sanctions hits — to add depth to an examination. This cuts against a line the team already drew deliberately: ADR-0009/0011 kept even EDM (the bank's own regulator-held, authoritative fraud data) as context only, never a basis for automated cross-checking, because the data wasn't reliable enough for that role. Scraped news/forum/social content is categorically weaker than EDM was. `CONTEXT.md`'s definition of **Finding** (a validated, human-signed-off, citation-grounded compliance gap) has no room for an unverified external source as an input today.

## Decision

Split into two proposals with materially different risk profiles, not one feature.

1. **Sanctions/watchlist screening against official government lists (OFAC, UN, EU, UK OFSI) — in scope as a near-term addition.** Real MCP servers exist today that screen names against these lists live from primary regulatory sources, each hit returning the program, legal basis, and a link to the official record. This is structurally close to the Notice Corpus's own trust model — authoritative, government-published, citable — and is arguably citation-grade in the same sense a notice clause is, unlike news or complaint content. If built: a new read-only tool (same shape as `edm.query()`) available to Compliance Analyst or Findings Author, not a standalone agent.

2. **News/adverse-media/consumer-complaint monitoring — explicitly deferred, not built.** Adverse-media screening is a real, mature compliance-industry category, but with known false-positive rates (35–95% depending on vendor/matching approach) and real legal exposure: ToS breach-of-contract risk, and consumer-complaint content is third-party personal data — GDPR/PDPL-type exposure for a bank regulator's own tooling to handle without a clear lawful basis. **This is deferred pending an actual answer from CBUAE's legal/compliance function on whether FPSD is even authorized to build external-media monitoring into its own tooling** — a legal question, not an engineering one.

## Consequences

Building ahead of the legal answer for proposal 2 risks real rework. If it's ever approved and built, it must surface as a distinctly-labeled, separate advisory signal — its own state field, never written into `compliance_verdicts[]` or `findings[]`, never blended with a grounded citation — consistent with ADR-0003's augmentation stance. Its data would also need retention/access treatment consistent with ADR-0024's pattern.

No agent, tool, or diagram change is made by this ADR. It records the scope decision; the sanctions-screening tool contract (proposal 1) remains an open item, not yet specified.
