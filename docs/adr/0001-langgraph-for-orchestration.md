# Use LangGraph for agent orchestration

The pipeline is augmentation, not autonomous (ADR-0003): it needs human-in-the-loop interrupt/checkpoint behavior at multiple points (clarification-question sign-off, findings/severity sign-off, before Assistant Governor submission). We chose **LangGraph** over other agent-orchestration frameworks specifically because its graph model gives granular, explicit control over interrupt/resume points, which fits this requirement more directly than frameworks built around a single autonomous agent loop.
