# v1 covers all three examination phases, lean, rather than one phase in depth

We initially scoped v1 to pre-examination intake tracking and gap analysis only (highest tedium/error today), with the examination-meeting and post-examination phases designed at a lighter level of detail but not built. This was reversed: v1 now spans **all three phases end-to-end** (pre-examination, examination meeting, post-examination — including transmittal letter and pre-exit deck drafting), but each phase gets only its bare-minimum slice rather than full depth.

This is worth recording because it's the opposite of the "narrow but deep" default a reader would expect from the phase-1 framing still visible in early design notes, and because it cascades: features that were deferred under the narrow scope (e.g. the severity/deadline rubric's editable UI) came back into v1 scope once the phase that consumes them was back in.

Concretely, "bare minimum" per phase means:
- **Pre-examination**: intake tracking (presence/format only, no content validation) + notice/clause compliance gap analysis (no EDM cross-checking).
- **Examination meeting**: agent drafts the pre-meeting clarification-question list from gap-analysis + EDM output; accepts human-typed meeting minutes post-meeting to extract further-submission action items. No audio/video transcription.
- **Post-examination**: findings/severity human sign-off, then transmittal letter drafted from a fixed template; pre-exit deck is a derived artifact off the same finding data, not independently authored. Assistant Governor approval is a status gate only (no active routing/notification); the rare exit-deck revision path is a deferred manual edge case.

License-type scope is separately narrowed to banks only for v1 (SVFs and others deferred to v2+), independent of this phase-scope decision.
