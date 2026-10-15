# Agentic Supervision: Roadmap & Implementation Next Steps

| Metadata | Details |
|---|---|
| **Document** | Roadmap & Implementation Next Steps |
| **Project** | LFI Examination Pipeline (CBUAE / FPSD) |
| **Status** | Approved for Implementation Planning |
| **Target Architecture** | v1 Lean End-to-End (Bank LFIs, English Only, On-Premise) |
| **Date** | October 2026 |
| **Companion Docs** | [PRD.md](PRD.md), [TDD.md](TDD.md), [agent-specifications.md](architecture/agent-specifications.md), [tool-contracts.md](architecture/tool-contracts.md) |
| **Live Portal** | [https://docs.codegraphlabs.com](https://docs.codegraphlabs.com) |

---

## 1. Executive Summary & Current State

The **Agentic Supervision** pipeline augments the Fraud Prevention and Supervision Department (FPSD) at CBUAE during compliance examinations of Licensed Financial Institutions (LFIs). 

As of October 2026:
* **The architectural design is complete and fully reconciled**: 22 design rounds, 138 numbered decisions, 32 ADRs, and 26 interactive C4/workflow/sequence diagrams.
* **All 4 independent design and Principal Engineer reviews have closed**: All Critical and High blocking items (including Pre-AG sign-off, redraft loops, audit log hash-chain schema, state schema fields, and intake triggers) are resolved and verified.
* **No application code has been written yet**: The next phase is engineering kickoff and implementation according to the dependency-driven build order.

---

## 2. Immediate Phase 0: Stakeholder Alignment & Prerequisites

Before code implementation starts, several organizational and infrastructure dependencies must be confirmed with CBUAE IT and FPSD leadership.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   PHASE 0: PREREQUISITES & ALIGNMENT                   │
├──────────────────────────┬──────────────────────────┬──────────────────┤
│ CBUAE IT & Security      │ FPSD Leadership          │ Examiner Team    │
├──────────────────────────┼──────────────────────────┼──────────────────┤
│ • Container platform     │ • Formally sign off on   │ • Supply annual  │
│   (VMs vs. OpenShift)    │   PRD.md / TDD.md        │   exam volume    │
│ • SSO / IdP choice       │ • Confirm English-only   │ • Time 2-3 manual│
│   (Azure AD vs. on-prem) │   scope for v1 (ADR-0031)│   exams for      │
│ • Secrets Manager choice │ • Define AI model-risk   │   baseline ROI   │
│ • Model candidate select │   sign-off procedure     │   metrics        │
│   (Qwen Coder/gpt-oss)   │                          │                  │
└──────────────────────────┴──────────────────────────┴──────────────────┘
```

### Action Items for Phase 0

1. **Host an IT Architecture Alignment Session (CBUAE IT)**:
   * **Platform**: Decide between bare-metal VMs vs. OpenShift/Kubernetes for container hosting.
   * **Model Selection**: Select the in-tenant open-weight model candidate (**Qwen Coder 32B/72B** vs. **gpt-oss-120b**) and provision initial GPU test hardware (NVIDIA H100/A100 instances).
   * **Authentication**: Confirm OIDC/SAML federated identity integration against CBUAE SSO.
   * **Secrets Management**: Select enterprise secrets store (HashiCorp Vault, CyberArk, or native K8s secrets).
2. **Confirm Formal Scope Approvals (FPSD Leadership)**:
   * **Formal Sign-off**: Record document approval for [PRD.md](PRD.md) and [TDD.md](TDD.md).
   * **Language Scope ([ADR-0031](adr/0031-english-only-v1-language-scope.md))**: Obtain explicit written confirmation that v1 targets English notices and submissions only, deferring Arabic OCR/normalization.
   * **Storage Sizing Validation ([ADR-0032](adr/0032-fraud-database-stores-document-content.md))**: Update Oracle storage provisioning assumptions to accommodate full document file retention.
3. **Capture Current-State Operational Baseline (Examiner Team)**:
   * Record average examiner hours spent on manual RFI document tracking and clause-by-clause review across 2–3 active manual examinations. This provides the baseline for measuring G1 success metrics post-launch.

---

## 3. Engineering Implementation: Phased Build Order

Per [TDD.md §8.1](TDD.md), implementation follows a strict dependency order: foundation data stores and adapters first, followed by deterministic services, guarded LLM agents, UI, and observability.

```
       ┌────────────────────────────────────────────────────────┐
       │   Sprint 1–2: Foundation Data Stores & Ingestion      │
       │   • Oracle Fraud DB Schema                             │
       │   • Ingestion Adapter (SFTP)                           │
       │   • Notice Corpus OCR & Segmentation Service           │
       └───────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │   Sprint 3: Deterministic Rule-Based Intake Tracker    │
       │   • RFI Question Parser                                │
       │   • Directory Polling & Submission Status Matrix       │
       │   • all_submitted & round_submitted(N) Events          │
       └───────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │   Sprint 4–5: Model Interface & Compliance Analyst    │
       │   • Model-Agnostic LLM Client (mTLS, in-tenant)        │
       │   • Exact Citation Grounding Hard Rule (ADR-0004)      │
       │   • Gap Analysis Node & Supervision Qs Drafting        │
       └───────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │   Sprint 6–7: Meeting Facilitator & Findings Author    │
       │   • Supervision Question Sign-off Gate (Interrupt 2)   │
       │   • Minutes Extraction & Further Submission Loop       │
       │   • Qualitative + Quantitative Tracks (ADR-0011)       │
       │   • Severity Rubric Service (ADR-0020)                 │
       └───────────────────────────┬────────────────────────────┘
                                   │
                                   ▼
       ┌────────────────────────────────────────────────────────┐
       │   Sprint 8–9: Approval Coordinator & UI Integration    │
       │   • Pre-AG Sign-off & Routine AG Revision Loops        │
       │   • Side-by-Side AG Feedback Review (Interrupt 8)      │
       │   • Examiner Dashboard (7 Tabs, RBAC-Gated Actions)    │
       │   • Hash-Chained Audit Trail & Langfuse Wiring         │
       └────────────────────────────────────────────────────────┘
```

### Sprint Breakdown & Work Packages

#### Sprints 1–2: Foundation Data Stores & Ingestion Layer
* **Work Package 1.1 — Oracle Fraud Database Schema**:
  * Implement tables for `notice_corpus`, `rfi_store`, `audit_trail`, and `analysis_output` (ADR-0012, ADR-0014).
  * Implement full document content binary/BLOB storage with at-rest encryption (ADR-0019, ADR-0032).
  * Implement hash-chained audit log schema and verification queries (`log_checkpoint` contract).
* **Work Package 1.2 — Ingestion Adapter Interface**:
  * Build the pluggable `IngestionAdapter` interface (`list_documents`, `get_metadata`, `fetch_document`) with SFTP provider (ADR-0006).
  * Implement retry with exponential backoff and distinct `tool_unavailable` vs. `no_match` error signaling (ADR-0018).
* **Work Package 1.3 — Notice Corpus Ingestion & Segmentation**:
  * Build the PDF parser, OCR pipeline (Tesseract/PaddleOCR), and clause-segmentation tool (`ocr_segment`).
  * Implement `notice_corpus.search()` and deterministic `notice_corpus.get_clause()` exact-match lookup (ADR-0027).
  * Build `supersession_graph.check()` relational lookup.

#### Sprint 3: Deterministic Intake Tracker (Rule-Based)
* **Work Package 2.1 — RFI Parser & Storage**:
  * Excel/Word RFI upload parser (`rfi_parser.parse`) extracting question ID, topic, and cited notice/clause (decision 85).
* **Work Package 2.2 — Submission State Tracker**:
  * Polling engine checking uploaded documents against fixed bank directory conventions.
  * Emit `all_submitted` event when round-1 documents are complete.
  * Emit `round_submitted(round: N)` event when round-N further submissions complete (decision 127).

#### Sprints 4–5: Model Interface & Compliance Analyst Agent
* **Work Package 3.1 — Model-Agnostic LLM Client**:
  * Build client interface targeting vLLM/TGI server endpoints via mTLS (ADR-0013, ADR-0019).
  * Add structured JSON validation with schema repair.
* **Work Package 3.2 — Compliance Analyst & Hard Citation Guardrail**:
  * LangGraph subgraph for batch gap analysis (ADR-0001).
  * Implement the non-negotiable **Hard Citation Grounding Rule** (ADR-0004): exact-match citation verification; emit `insufficient_grounding, needs human review` if grounding fails.
  * Generate initial supervision-question drafts.

#### Sprints 6–7: Meeting Facilitator & Findings Author
* **Work Package 4.1 — Meeting Facilitator**:
  * Phase 1 handoff: Supervision questions finalization and per-question `interrupt()` sign-off gate (HITL Checkpoint 2).
  * Phase 2 meeting minutes text ingestion and action-item extraction (`further_submission_requests[]`, HITL Checkpoint 3).
  * Human-judged sufficiency check gate (HITL Checkpoint 4).
* **Work Package 4.2 — Findings Author & Deterministic Services**:
  * Deterministic arithmetic service: `quant_analysis.compute` over EDM quarterly figures (ADR-0021).
  * Deterministic rubric service: `severity_rubric.lookup` with code-enforced equality check (ADR-0017, ADR-0020).
  * Dual-track findings generation (qualitative clause compliance + quantitative EDM context, never reconciled, ADR-0009/0011).
  * Findings per-item sign-off gate (HITL Checkpoint 5).
  * Transmittal letter and pre-exit deck template filler (`template_fill`, Word/PowerPoint generation).

#### Sprints 8–9: Approval Coordinator, Dashboard & Governance
* **Work Package 5.1 — Approval Coordinator & Routine Revision Loops**:
  * Pre-AG Sign-off checkpoint gating transmittal letter delivery (HITL Checkpoint 6, ADR-0003).
  * AG Feedback Review node displaying raw feedback side-by-side with editable AI summary (HITL Checkpoint 8).
  * Automatic routing of AG or Pre-Exit revisions back through `draft_findings` and re-signoff (ADR-0010, decision 123).
* **Work Package 5.2 — Examiner Dashboard UI**:
  * Unified web frontend: Portfolio Home + 7 Tabbed Workspaces (Setup, Intake, Gap Analysis, Meeting, Findings, Approval, Audit Log).
  * RBAC action-gating for `Examiner`, `Lead/Approver`, and `Auditor/Compliance Reviewer` (ADR-0008, ADR-0029).
  * Optimistic concurrency with etag checking on checkpoint resolution (ADR-0018).
* **Work Package 5.3 — Full Observability & Audit Wiring**:
  * Wire OpenTelemetry tracing and Langfuse prompt/completion tracking (ADR-0016).
  * Calculate live **human-agreement rate** metric across all 9 checkpoints.

---

## 4. Launch & Rollout Plan

To ensure regulatory reliability, deployment proceeds through a multi-stage pilot with strict go/no-go gates.

```
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│   SHADOW MODE   │  ──►  │   LIVE PILOT    │  ──►  │  FULL ROLLOUT   │
│  (1–2 active    │       │  (2–3 live      │       │  (Bank-wide     │
│   examinations) │       │   examinations) │       │   examinations) │
└─────────────────┘       └─────────────────┘       └─────────────────┘
  • Pipeline runs           • Lead/Approver signs     • Standard
    offline in parallel       off in system             operating model
  • Examiners do manual     • Manual fallback ready   • Continuous
    work as usual           • Track agreement rate      monitoring
  • Compare outputs           and latency
```

### Go/No-Go Gate Criteria

Before moving from Live Pilot to Full Production, the system must satisfy four non-negotiable criteria:
1. **Zero Grounding Violations (Hard Gate)**: 0 verified instances of a compliance verdict citing a nonexistent or misquoted clause (ADR-0004).
2. **Healthy Human-Agreement Rate**: Lead/Approver acceptance rate stays within an acceptable band (e.g. 70%–92%). A rate near 100% flags automation bias (ADR-0016).
3. **Audit Log Integrity**: Zero unverified `chain_integrity_violation` alerts in the append-only audit trail.
4. **SLA Adherence**: No checkpoint SLA breach caused by pipeline error or hanging inference.

---

## 5. Summary Reference Map

* **Architecture Diagrams**: [docs.codegraphlabs.com](https://docs.codegraphlabs.com)
* **Product Requirements**: [PRD.md](PRD.md)
* **Technical Design**: [TDD.md](TDD.md)
* **Agent Specifications**: [agent-specifications.md](architecture/agent-specifications.md)
* **Tool Contracts**: [tool-contracts.md](architecture/tool-contracts.md)
* **Decision History**: [decision-log.md](decision-log.md)
