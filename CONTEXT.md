# LFI Examination Pipeline

FPSD (Fraud Prevention and Supervision Department) at CBUAE runs periodic compliance examinations of Licensed Financial Institutions. This context is a multi-agent pipeline that augments (not replaces) the examiner team across the three phases of an examination: pre-examination intake and gap analysis, the examination meeting, and post-examination findings/reporting.

## Language

### Institutions and examination structure

**LFI (Licensed Financial Institution)**:
A regulated entity CBUAE examines for compliance. Has a **license type** (e.g. bank, SVF) that determines which RFI template and folder structure applies to it.
_Avoid_: institution, entity, client

**License type**:
The regulatory category an LFI falls under (bank, SVF, etc.). Determines which RFI question bank and folder structure apply — all LFIs of the same license type share both.
_Avoid_: LFI type, category

**Examination**:
One instance of FPSD reviewing a specific LFI for compliance, running through the pre-examination, examination meeting, and post-examination phases.
_Avoid_: audit, review, engagement

**Examiner**:
A member of the FPSD team who drafts RFI responses review, gap analysis, supervision questions, and findings. Distinct from the **lead/approver** role, which signs off on an examiner's drafts at each human-in-the-loop checkpoint.
_Avoid_: reviewer, analyst, user

**Lead / approver**:
The role that signs off on drafted output at each human sign-off checkpoint (supervision-question list, findings/severity, before Assistant Governor submission). Distinct from the examiner role that produces the drafts.
_Avoid_: senior examiner, manager

### Pre-examination

**RFI (Request for Information)**:
The set of 15-20 questions sent to an LFI 30 days before an examination starts, each tagged with the notice/standard/clause it relates to. Delivered as a structured spreadsheet (Question ID, Question text, Notice reference, Clause reference) — not free text.
_Avoid_: questionnaire, request list

**Shared workspace (CB Workspace)**:
The LFI-facing upload location where an LFI submits documents against RFI questions during the 30-day window, structured by the fixed per-license-type folder structure. Reached by the pipeline over SFTP via the ingestion adapter, not accessed directly. **Being delisted** — Data Office is replacing it with the **Smart Portal**.
_Avoid_: portal, upload folder

**Smart Portal**:
The internally-developed replacement for the shared workspace, reachable via API or SFTP. Not yet built; the ingestion adapter is designed so swapping it in for the shared workspace requires no downstream changes.
_Avoid_: new portal, API portal (it supports both API and SFTP, not API-only)

**Ingestion adapter**:
The pluggable interface (list documents / fetch document / get last-modified) the pipeline uses to read from the shared workspace or Smart Portal, isolating downstream logic from the concrete transport.
_Avoid_: sync job, connector

**Submitted (intake status)**:
A document's presence status for a given RFI question: the file exists, is non-empty, and matches the expected filename/format convention. Distinct from a document being judged a *plausible* answer to the question — that is a separate, later concern (content validation), not part of "submitted."
_Avoid_: uploaded, complete, answered

**Notice**:
A CBUAE-issued regulatory document (standard/law) containing clauses that LFIs must comply with. Source corpus for every compliance verdict's citation. May exist only as a scanned image, requiring OCR before it is queryable.
_Avoid_: standard, regulation, circular (unless that's the notice's own type name)

**Clause**:
A single addressable provision within a notice — the unit a compliance verdict cites and the unit an RFI question is tagged against.
_Avoid_: section, provision, requirement

**Supersession**:
A relationship where a clause in one notice overrides a clause in another notice under certain conditions. Modeled as a dependency graph derived from the notice corpus; only high-uncertainty supersessions are escalated to human review.
_Avoid_: override, replacement, precedence

**Compliance verdict**:
The pipeline's judgment (compliant / non-compliant / etc.) on whether a submitted document satisfies a clause. Must cite the exact notice and exact clause sentence verbatim; if no high-confidence grounded citation exists, the verdict is "insufficient grounding, needs human review" instead of a guess.
_Avoid_: finding (finding is the post-examination, human-validated artifact — see below), result, assessment

**Gap analysis**:
The batch step — triggered once intake is complete for all RFI questions, not run incrementally during the 30-day window — that produces compliance verdicts for submitted documents against their tagged clauses, using notice-corpus citations and EDM data as complementary context. Its output is three artifacts: **supervision questions**, **LFI gaps and findings**, and **areas of strengths and weaknesses**.
_Avoid_: review, analysis

**EDM (Enterprise Data Management)**:
The Oracle-based system holding an LFI's quarterly fraud submissions — business volume and fraud-transaction figures for cards and transfers, among other fields. In v1 it (a) complements document evidence as context during gap analysis, (b) is an input to drafting supervision questions, and (c) is analyzed on its own terms as **quantitative analysis** (see below) — but its figures are never cross-checked for consistency against document claims.
_Avoid_: data warehouse

**Fraud Database**:
The umbrella data store the pipeline owns: it holds the notice corpus, the audit trail, and all AI-generated analysis/interpretation output, and replicates the source system's file/folder structure. A physical-storage grouping — the logical distinctions between notice corpus, audit trail, and analysis output (below) still hold for how the pipeline reasons about and cites data.
_Avoid_: analysis store, AI database

### Examination meeting

**Supervision question**:
A question drafted from gap analysis (and EDM context) — used both as a general Phase-1 output and, specifically, as the list prepared for the live examination meeting. The meeting-ready list requires human sign-off before use.
_Avoid_: clarification question, follow-up question

**Meeting minutes**:
The human-authored notes/record of the examination meeting, submitted as text input to the pipeline afterward so it can extract further-submission action items.
_Avoid_: meeting notes, transcript (transcript implies audio/video capture, which is out of scope)

**Further submission**:
An additional document an LFI must provide as a result of the examination meeting, submitted through the same shared workspace mechanism as the original RFI documents.
_Avoid_: follow-up document, post-meeting submission

### Post-examination

**Findings analysis**:
The Phase-2 step (after the examination meeting's documents/answers are judged sufficient) that produces **findings** along two parallel tracks: **qualitative analysis** (notice/clause compliance narrative — citations and verdicts, per the grounding rule) and **quantitative analysis** (EDM figures analyzed on their own terms — volumes, loss amounts, trends/ratios — not cross-checked against documents). Distinct from **gap analysis** (Phase 1, pre-meeting).
_Avoid_: analyze findings, review

**Finding**:
A validated, human-signed-off compliance gap that will appear in the transmittal letter, carrying a severity and a deadline. Distinct from a compliance verdict, which is the pipeline's unvalidated draft judgment before human sign-off.
_Avoid_: issue, gap, non-compliance item

**Severity**:
A finding's rated importance — High / Medium / Low — assigned via the severity/deadline rubric.
_Avoid_: priority, risk rating

**Severity/deadline rubric**:
The lookup table mapping severity to a remediation deadline. Coded as data, editable through a UI rather than hardcoded, since it changes independently of the pipeline's code.
_Avoid_: policy table, severity matrix

**Transmittal letter**:
The formal document listing all findings (each with severity and deadline) sent toward Assistant Governor approval, filled into a fixed existing Word template.
_Avoid_: findings letter, report

**Pre-exit deck**:
The presentation deck derived from the same finding data as the transmittal letter, filled into a fixed existing PPT template, for presentation to LFI leadership after Assistant Governor approval.
_Avoid_: findings deck, presentation

**Exit deck**:
The revised version of the pre-exit deck (and its accompanying transmittal letter), produced when the LFI raises concerns at the pre-exit meeting. **Routine, not rare** — modeled in v1 as a normal revision loop: LFI raises concerns → concerns noted → letter/deck updated → re-checked → proceed to exit meeting.
_Avoid_: revised deck, final deck

**Assistant Governor (AG) approval**:
The sign-off an Assistant Governor gives to the transmittal letter and pre-exit deck before they reach LFI leadership. Modeled in v1 as real pipeline state with a revision loop — showcase to AG → AG requests changes → back to findings/letter/deck drafting → re-showcase — not just a flat status field. The actual showcase/approval meeting itself happens outside the pipeline; only the resulting state (approved / changes requested) is tracked.
_Avoid_: AG sign-off, final approval

### Cross-cutting

**Human-in-the-loop checkpoint**:
One of the points where a lead/approver must sign off before the pipeline proceeds: the supervision-question list, findings/severity, before Assistant Governor submission, and — per the AG and pre-exit revision loops — each time a revision is requested and redrafted. The pipeline is augmentation, not autonomous, specifically at these points.
_Avoid_: gate (used for the narrower AG-approval status gate specifically), approval step

**Audit trail**:
The append-only record, captured at every human-in-the-loop checkpoint, of what the pipeline proposed versus what a human approved or overrode.
_Avoid_: activity log, history
