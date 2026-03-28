# Agent States Specification

## Overview

The Regulatory Intelligence Agent operates as a deterministic finite-state machine. Each state has a single responsibility, defined entry conditions, internal actions, and explicit exit transitions.

---

## State Index

| # | State | Type | Description |
|---|---|---|---|
| 1 | DISCOVERY | Automated | Continuous polling of HA sources |
| 2 | INGESTION | Automated | Document retrieval and normalization |
| 3 | INTELLIGENCE_ANALYSIS | Automated (AI) | Classification, impact mapping, risk scoring |
| 4 | HUMAN_REVIEW | **Human Gate** | Mandatory human validation of AI output |
| 5 | ORCHESTRATION | Automated | Downstream task execution |
| 6 | ACTION_TRACKING | Automated | Workflow monitoring and feedback collection |
| 7 | LEARNING_LOG | Automated | Rejection logging and model improvement input |

---

## State 1: DISCOVERY

**Purpose:** Maintain awareness of the HA regulatory landscape by continuously monitoring known sources.

**Entry Condition:** System startup, or reset from ACTION_TRACKING / LEARNING_LOG.

**Actions:**
- Poll configured URL endpoints and RSS feeds at a defined interval (default: every 2 hours).
- Retrieve document metadata (title, publication date, document ID, URL).
- Compare metadata and checksums against `Known_Guidance_DB`.
- Log each poll attempt with timestamp and source status.

**Exit Transition:**
- → **INGESTION** if a new or updated document is detected (checksum mismatch or new record).
- → **DISCOVERY** (self-loop) if no change is detected.

**Error Handling:**
- If a source is unreachable: log failure, retry after backoff interval, alert ops after 3 consecutive failures.

---

## State 2: INGESTION

**Purpose:** Retrieve the full document and produce a clean, machine-readable text representation.

**Entry Condition:** New/updated document identified in DISCOVERY.

**Actions:**
- Download source document (PDF or HTML).
- Apply OCR pipeline for PDF documents.
- Strip boilerplate content (headers, footers, page numbers, legal notices).
- Segment text into structured sections (title, scope, effective date, body paragraphs).
- Store normalized text and raw source in document store with a unique `doc_id`.

**Exit Transition:**
- → **INTELLIGENCE_ANALYSIS** on successful normalization.
- → **DISCOVERY** (with error log) after 3 failed ingestion retries.

**Error Handling:**
- OCR confidence below threshold: flag for manual ingestion review.
- Corrupt or password-protected PDF: escalate immediately to human operator.

---

## State 3: INTELLIGENCE_ANALYSIS

**Purpose:** Apply AI reasoning to interpret the regulatory document and assess its business impact.

**Entry Condition:** Clean, normalized document text available from INGESTION.

**Actions:**

1. **Classification**
   - Tag the document with one or more categories: `Safety`, `Labeling`, `Manufacturing`, `Clinical`.
   - Record confidence score per tag.

2. **Impact Mapping**
   - Cross-reference guidance content against internal `Product_Registry` and `SOP_Registry`.
   - Identify which products, therapeutic areas, or SOPs are potentially affected.
   - Generate a structured impact list with justification per item.

3. **Risk Scoring**
   - Assign an overall risk level: `High`, `Medium`, or `Low`.
   - Risk score is derived from: classification category, number of affected products, regulatory deadline (if stated), and geographic scope.

4. **AI Summary Generation**
   - Produce a concise human-readable summary (max 300 words).
   - Include: document purpose, key changes, affected scope, recommended actions.

**Exit Transition:**
- → **HUMAN_REVIEW** (always; no bypass permitted).

---

## State 4: HUMAN_REVIEW *(Critical Boundary)*

**Purpose:** Ensure no downstream action is taken without explicit human authorization. This state is the sole mandatory human gate in the pipeline.

**Entry Condition:** AI analysis package (summary, classifications, impact list, risk score) is complete.

**Actions:**
- Present a structured review interface showing:
  - AI-generated summary and classifications
  - Side-by-side view of source text excerpts
  - Affected product/SOP impact list
  - Risk score with rationale
- Accept one of three human decisions:
  - **Approve:** AI output accepted as-is.
  - **Modify:** Human edits summary, classification, impact list, or risk score. Modified version is recorded.
  - **Reject:** AI output deemed incorrect or irrelevant.
- Log reviewer identity, timestamp, decision type, and any free-text rationale.

**Exit Transition:**
- → **ORCHESTRATION** on `Approve` or `Modify`.
- → **LEARNING_LOG** on `Reject`, then reset to **DISCOVERY**.

**SLA:**
- High Risk: human review expected within 48 hours.
- Medium/Low Risk: human review expected within 5 business days.
- Escalation alert triggered if SLA is breached.

---

## State 5: ORCHESTRATION

**Purpose:** Execute all approved downstream compliance and communication tasks.

**Entry Condition:** Human-approved (or modified) analysis package.

**Actions:**
- Send alert notifications via configured channels (Microsoft Teams, Email).
- Create a Change Control draft record in the QMS platform via API.
- Notify designated stakeholder groups (Medical Affairs, QA, Regulatory Affairs).
- Attach AI summary, source document link, and reviewer decision to all outgoing items.
- Record all triggered actions with timestamps in `Action_Log`.

**Exit Transition:**
- → **ACTION_TRACKING** once all tasks are dispatched.

**Error Handling:**
- QMS API failure: retry 3 times, then hold and alert the integration ops team.
- Notification failure: log and retry; do not block state transition.

---

## State 6: ACTION_TRACKING

**Purpose:** Monitor the lifecycle of initiated workflows and collect quality feedback.

**Entry Condition:** All orchestration tasks dispatched.

**Actions:**
- Poll QMS for change control status (Open → In Review → Closed).
- Collect stakeholder feedback on alert relevance and accuracy.
- Record final resolution status per action item.
- Compute and log metrics: detection-to-review time, review-to-action time, false positive flag (if stakeholders mark alert as not applicable).

**Exit Transition:**
- → **DISCOVERY** after metrics are logged (cycle complete).

---

## State 7: LEARNING_LOG

**Purpose:** Capture rejected AI outputs to support continuous model improvement.

**Entry Condition:** Human reviewer selects `Reject` in HUMAN_REVIEW.

**Actions:**
- Store the full AI analysis package (classifications, summary, impact list, risk score) tagged as `rejected`.
- Store reviewer's free-text rationale.
- Increment rejection counter for the source document type and classification category.
- Flag record for periodic model fine-tuning or prompt engineering review.

**Exit Transition:**
- → **DISCOVERY** (pipeline resets; no downstream actions triggered).
