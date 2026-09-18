# Use LangGraph for agent orchestration

## Status

Accepted — 2026-09-15

## Context

The pipeline is augmentation, not autonomous (ADR-0003): it needs human-in-the-loop interrupt/checkpoint behavior at multiple points (supervision-question sign-off, findings/severity sign-off, before Assistant Governor submission).

## Decision

We chose **LangGraph** over other agent-orchestration frameworks specifically because its graph model gives granular, explicit control over interrupt/resume points, which fits this requirement more directly than frameworks built around a single autonomous agent loop.

## Consequences

Every agent in the pipeline is built as a LangGraph node (or small subgraph), and every HITL checkpoint is implemented as a LangGraph `interrupt()`. This shapes the failure/retry/idempotency semantics later defined in ADR-0018 and the state-schema versioning requirement in ADR-0025, both of which assume durable, resumable graph state as a given.
