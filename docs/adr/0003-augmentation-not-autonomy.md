# Pipeline is augmentation/co-pilot, never autonomous, at three specific checkpoints

Given the pipeline produces regulatory findings against a regulated financial institution, we decided against full autonomy anywhere output leaves the pipeline's draft state and becomes something acted on. A human (the lead/approver role) must sign off at exactly three points: (a) the final supervision-question list before the examination meeting, (b) final findings/severity before the transmittal letter is drafted, and (c) before anything is submitted for Assistant Governor approval.

This is a deliberate, hard-to-reverse boundary, not an incidental MVP limitation — it shaped downstream decisions including the citation-grounding hard rule (ADR-0004), the RBAC model (examiner vs. lead/approver), and the requirement for an audit trail of proposed-vs-approved state at each checkpoint. Removing a checkpoint later would be a regulatory-posture change, not a feature flag flip.
