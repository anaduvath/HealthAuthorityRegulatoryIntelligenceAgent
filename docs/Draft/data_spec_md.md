# Data Specification

This document defines all persistent data stores, reference registries, and data models used by the Regulatory Intelligence Agent.

---

## 1. Known_Guidance_DB

**Purpose:** Master registry of all previously seen regulatory documents. Used by DISCOVERY to detect changes.

**Storage type:** Relational (e.g., PostgreSQL) or document store (e.g., MongoDB)

| Field | Type | Description |
|---|---|---|
| `record_id` | UUID | Internal primary key |
| `ha` | enum: FDA, EMA, MHRA | Health Authority source |
| `document_id` | string | HA-assigned document identifier |
| `title` | string | Document title |
| `url` | string | Source URL |
| `publication_date` | date | Date published by HA |
| `checksum` | string (SHA-256) | Hash of document content for change detection |
| `first_seen_at` | datetime | Timestamp of initial detection |
| `last_checked_at` | datetime | Timestamp of last successful poll |
| `status` | enum: Active, Superseded, Withdrawn | Current document lifecycle status |

---

## 2. Document_Store

**Purpose:** Stores raw source files and normalized text outputs from INGESTION.

**Storage type:** Object storage (e.g., Azure Blob, S3) for raw files; document store for structured text.

| Field | Type | Description |
|---|---|---|
| `doc_id` | UUID | Internal document identifier |
| `event_id` | UUID | Reference to DISCOVERY trigger event |
| `raw_source_path` | string | Path to original PDF/HTML file |
| `normalized_text` | object | Structured text with sections (see I/O contract) |
| `ocr_confidence_score` | float | OCR quality score (null for HTML sources) |
| `ingested_at` | datetime | Ingestion completion timestamp |
| `source_metadata` | object | HA, URL, title, publication date |

---

## 3. Analysis_Store

**Purpose:** Persists all AI-generated analysis outputs from INTELLIGENCE_ANALYSIS, including those later rejected.

**Storage type:** Document store or relational DB with JSON column support.

| Field | Type | Description |
|---|---|---|
| `analysis_id` | UUID | Primary key |
| `doc_id` | UUID | Reference to Document_Store |
| `analysed_at` | datetime | Analysis completion timestamp |
| `classification` | array of objects | Categories with confidence scores |
| `impact_mapping` | array of objects | Affected entities with rationale |
| `risk_score` | object | Level, numeric score, rationale, factors |
| `ai_summary` | string | Generated human-readable summary |
| `model_version` | string | Version of AI model used |

---

## 4. Review_Log

**Purpose:** Audit trail of all HUMAN_REVIEW decisions. Immutable append-only log.

**Storage type:** Relational DB (append-only; no UPDATE or DELETE operations permitted)

| Field | Type | Description |
|---|---|---|
| `review_id` | UUID | Primary key |
| `analysis_id` | UUID | Reference to Analysis_Store |
| `reviewed_at` | datetime | Decision timestamp |
| `reviewer_user_id` | string | Identity of the reviewer |
| `reviewer_role` | string | Role of the reviewer |
| `decision` | enum: Approve, Modify, Reject | Human decision |
| `rationale` | string (nullable) | Free-text reviewer commentary |
| `approved_output` | object (nullable) | Final approved/modified content |
| `original_ai_output_preserved` | boolean | Always `true`; enforced by schema |

---

## 5. Action_Log

**Purpose:** Records all orchestration actions dispatched and their eventual resolution.

**Storage type:** Relational DB

| Field | Type | Description |
|---|---|---|
| `action_id` | UUID | Primary key |
| `orchestration_id` | UUID | Reference to orchestration run |
| `review_id` | UUID | Reference to Review_Log |
| `action_type` | enum: Alert, QMS_Draft, StakeholderNotification | Action category |
| `target` | string | System, channel, or recipient |
| `status` | enum: Sent, Failed, Pending, Completed | Current status |
| `dispatched_at` | datetime | Time action was triggered |
| `resolved_at` | datetime (nullable) | Time action reached terminal state |
| `external_reference_id` | string (nullable) | ID in target system (e.g., QMS draft ID) |

---

## 6. Learning_Log

**Purpose:** Captures rejected AI outputs for model improvement and audit.

**Storage type:** Document store (write-once)

| Field | Type | Description |
|---|---|---|
| `log_id` | UUID | Primary key |
| `review_id` | UUID | Reference to Review_Log |
| `doc_id` | UUID | Reference to Document_Store |
| `logged_at` | datetime | Timestamp of log creation |
| `rejection_rationale` | string | Human reviewer's explanation |
| `ai_output_snapshot` | object | Full copy of INTELLIGENCE_ANALYSIS output |
| `classification_categories_rejected` | array of strings | Which categories were disputed |
| `flagged_for_retraining` | boolean | Whether flagged for ML pipeline |

---

## 7. Product_Registry (Reference)

**Purpose:** Internal list of products used for impact mapping in INTELLIGENCE_ANALYSIS.

**Managed by:** Regulatory Affairs / Master Data Management team.

| Field | Type | Description |
|---|---|---|
| `product_id` | string | Internal product identifier |
| `product_name` | string | Commercial or development name |
| `therapeutic_area` | string | TA classification |
| `regulatory_markets` | array: FDA, EMA, MHRA | Markets where approved/pending |
| `development_stage` | enum: Marketed, Clinical, Preclinical | Current lifecycle stage |
| `relevant_guidance_categories` | array of strings | Which guidance types apply |

---

## 8. SOP_Registry (Reference)

**Purpose:** Internal list of Standard Operating Procedures used for impact mapping.

| Field | Type | Description |
|---|---|---|
| `sop_id` | string | Internal SOP identifier |
| `title` | string | SOP title |
| `category` | enum: Safety, Labeling, Manufacturing, Clinical | Mapped category |
| `owner` | string | Functional team owner |
| `version` | string | Current version |
| `last_reviewed` | date | Last review date |

---

## 9. Metrics_Store

**Purpose:** Aggregated performance metrics per agent cycle, used for dashboards and continuous improvement.

| Field | Type | Description |
|---|---|---|
| `cycle_id` | UUID | Unique cycle identifier |
| `doc_id` | UUID | Reference document |
| `detection_to_review_hours` | float | Time from detection to human review start |
| `review_to_action_hours` | float | Time from approval to orchestration dispatch |
| `end_to_end_hours` | float | Total cycle duration |
| `decision_type` | enum: Approve, Modify, Reject | Human decision outcome |
| `false_positive_flagged` | boolean | Stakeholder-flagged false positive |
| `risk_level` | enum: High, Medium, Low | Risk level of the document |
| `ha_source` | enum: FDA, EMA, MHRA | Source Health Authority |
| `logged_at` | datetime | Metrics finalization timestamp |

---

## Data Retention Policy

| Store | Retention Period | Notes |
|---|---|---|
| Document_Store (raw files) | 10 years | Regulatory requirement |
| Analysis_Store | 10 years | Linked to raw documents |
| Review_Log | 10 years | Audit trail; append-only |
| Action_Log | 7 years | Compliance actions |
| Learning_Log | 5 years | Model training data |
| Metrics_Store | 3 years | Operational analytics |
