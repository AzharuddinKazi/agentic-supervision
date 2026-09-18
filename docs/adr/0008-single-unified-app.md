# One unified web app, not per-feature standalone tools

## Status

Accepted — 2026-09-15

## Context

v1 has several UI touchpoints across the three phases (intake tracker, rubric editor, gap-analysis review, supervision-question review, meeting action-item review, findings sign-off, AG-approval status gate). We considered shipping these as separate, loosely-connected screens or tools stitched together only by the underlying LangGraph state, since each feature's review surface was already decided to be self-contained (per-feature review UX, not a unified review queue).

## Decision

We instead committed to one unified app that examiners live in throughout the exam cycle, built comprehensively (every in-scope touchpoint fully built, not a spreadsheet-export stopgap) even though the pipeline's feature scope underneath stays lean.

## Consequences

This is worth recording because "per-feature review surfaces" (a related, already-settled decision) could easily be read as license to also scatter the UI itself across disconnected tools — that's explicitly not the case. Fragmenting the UI would undercut day-to-day adoption of the augmentation model (ADR-0003) more than it would save build effort, so it's one of the few places we chose not to cut corners even in a bare-minimum v1. (ADR-0029 later settles this app's actual internal structure — navigation, tabs, RBAC-gated actions.)
