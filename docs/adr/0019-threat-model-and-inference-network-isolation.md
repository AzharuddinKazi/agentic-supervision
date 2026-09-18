# Threat model and network isolation for the LLM Inference Service

## Status

Accepted — 2026-09-15

## Context

An independent architecture review (`.scratch/architecture-review-2026-09-15.md`, High Finding #5) found that ADR-0013's self-hosting rationale — "no examination data leaves the enterprise boundary at inference time" — was not backed by anything in the deployment diagram beyond one undifferentiated "On-prem deployment region" boundary. The LLM Inference Service sees every document, every meeting minute, and every finding in plaintext at inference time; a container in the same network segment with no additional isolation is a materially weaker boundary than ADR-0013's framing implies. Nothing addressed encryption-at-rest for the Fraud Database, or secrets management for the SFTP credentials, Oracle connection string, and service-to-service auth tokens the pipeline already implicitly depends on.

## Decision

We're closing three concrete gaps, all fail-closed defaults rather than aspirational goals:

1. **Model/Inference security zone.** A dedicated `security-group` boundary wraps only `llm_inference` (previously it sat undifferentiated inside the general on-prem region alongside every other backend service). Network policy: only the four LLM agents (`intake_tracker` is rule-based and does not call the LLM) may open a connection to this zone, over mTLS, and nothing else — no direct examiner-app or external access.
2. **Encryption at rest for the Fraud Database.** ADR-0014 named the engine (Oracle) but was silent on encryption. Stated explicitly here: the Fraud Database (audit trail, notice corpus, findings, RFI store) uses Transparent Data Encryption (TDE) or equivalent at-rest encryption.
3. **A secrets-management component.** No vault/secret-manager existed anywhere in the deployment diagram — SFTP credentials, the Oracle connection string, and internal service-auth tokens had no documented home. A `secrets_manager` component is added to the deployment diagram, feeding the Ingestion Adapter (SFTP creds), `edm.query`'s Oracle connection, and the LLM Inference Service's mTLS certificates. Which product (HashiCorp Vault vs. an existing CBUAE enterprise secrets store) is explicitly **not decided here** — same open-item treatment as the deployment platform in ADR-0015.

## Consequences

A compromised agent container still can't be used as a pivot to anything *other* than the inference service it already legitimately calls; a compromised or misconfigured route elsewhere in the region can't reach the inference service at all. ADR-0013's "data never leaves the enterprise boundary" claim is a network-topology claim, not just a hosting-location claim — it is only true in the meaningful sense (limiting *lateral* exposure, not just external exposure) if the inference service is actually isolated from everything that doesn't need to call it. This ADR doesn't change ADR-0013's decision to self-host; it makes good on the isolation that decision's own rationale already assumed.

**What's still explicitly out of scope here** (flagged, not solved): a full threat model covering the ingestion path (malicious file upload from the LFI's Shared Workspace/Smart Portal), a formal STRIDE-style pass over every component, and CBUAE's own security team's sign-off process — scheduled for before go-live, not before implementation. This deferral does **not** cover prompt injection via LFI-authored content, which is a different threat addressed separately in ADR-0030.
