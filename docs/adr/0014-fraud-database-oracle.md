# Fraud Database runs on Oracle

ADR-0012 established the Fraud Database as one physical store for the notice corpus, audit trail, and analysis output. We're pinning its engine to **Oracle**, matching EDM's existing engine, rather than introducing a second database technology (e.g., PostgreSQL) into CBUAE's operational footprint for this one pipeline.

This is worth recording as a real trade-off: Oracle isn't the default choice one would reach for on workload fit alone (the notice corpus's per-clause text search and the audit trail's append-only log are both comfortably served by several engines) — it's chosen for operational consistency with EDM, meaning CBUAE's existing Oracle DBA expertise, backup/HA tooling, and licensing already cover this new store instead of standing up a second engine to operate.
