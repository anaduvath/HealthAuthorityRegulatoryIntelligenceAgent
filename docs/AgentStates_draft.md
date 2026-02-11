State Definitions

State: DISCOVERY
Description: Continuous monitoring of Health Authority (HA) sources (FDA, EMA, MHRA).
Actions: * Poll URL endpoints and RSS feeds.
Compare metadata/checksums against the Known_Guidance_DB.
Transition: Move to INGESTION if a new/updated document is detected.

State: INGESTION
Description: Data retrieval and structural normalization.
Actions: * Download PDF/HTML source.
Perform OCR and clean text (remove boilerplate).
Transition: Move to INTELLIGENCE_ANALYSIS once content is digitized.

State: INTELLIGENCE_ANALYSIS
Description: The "Cognitive" phase where the agent interprets the text.
Actions: * Classification: Tag as Safety, Labeling, Manufacturing, or Clinical.
Impact Mapping: Compare guidance against internal product lists and SOPs.
Risk Scoring: Assign a level (High, Medium, Low).
Transition: Move to HUMAN_REVIEW (Mandatory).

State: HUMAN_REVIEW (Critical Boundary)
Description: System waits for human validation of the AI’s interpretation.
Actions: * Present "AI Summary" vs "Source Text" to Regulatory/QA teams.
Await user input: Approve, Reject, or Modify.
Transition: * If Approve/Modify: Move to ORCHESTRATION.
If Reject: Move to LEARNING_LOG and reset to DISCOVERY.

State: ORCHESTRATION
Description: Execution of downstream administrative tasks.
Actions: * Trigger alerts (Teams/Email).
Initiate Change Control drafts in Quality Management Systems (QMS).
Notify relevant stakeholders (Medical Affairs, QA).
Transition: Move to ACTION_TRACKING.

State: ACTION_TRACKING
Description: Post-alert monitoring.
Actions: * Monitor status of initiated workflows.
Collect user feedback on alert accuracy.

Transition: Reset to DISCOVERY after logging metrics.
