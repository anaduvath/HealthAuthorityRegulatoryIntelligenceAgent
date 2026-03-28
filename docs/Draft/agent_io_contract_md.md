# Agent I/O Contract

This document defines the precise input and output schemas for each state transition in the Regulatory Intelligence Agent pipeline.

All payloads are JSON unless otherwise noted. Field names use `snake_case`. All timestamps are ISO 8601 UTC.

---

## DISCOVERY → INGESTION

### Input to DISCOVERY
```json
{
  "poll_config": {
    "sources": [
      {
        "ha": "FDA | EMA | MHRA",
        "url": "string (endpoint or RSS feed URL)",
        "poll_interval_minutes": "integer"
      }
    ],
    "known_guidance_db_endpoint": "string (URL or file path)"
  }
}
```

### Output of DISCOVERY (trigger payload to INGESTION)
```json
{
  "event_id": "string (UUID)",
  "detected_at": "datetime",
  "source": {
    "ha": "FDA | EMA | MHRA",
    "url": "string",
    "document_id": "string",
    "title": "string",
    "publication_date": "date",
    "checksum": "string (SHA-256)",
    "change_type": "NEW | UPDATED"
  }
}
```

---

## INGESTION → INTELLIGENCE_ANALYSIS

### Input to INGESTION
Trigger payload from DISCOVERY (above).

### Output of INGESTION
```json
{
  "doc_id": "string (UUID)",
  "event_id": "string (ref to DISCOVERY event)",
  "source_metadata": {
    "ha": "string",
    "original_url": "string",
    "document_id": "string",
    "title": "string",
    "publication_date": "date",
    "file_type": "PDF | HTML"
  },
  "ingested_at": "datetime",
  "normalized_text": {
    "full_text": "string",
    "sections": [
      {
        "section_id": "integer",
        "heading": "string",
        "body": "string"
      }
    ],
    "effective_date": "date | null",
    "geographic_scope": ["string"]
  },
  "ocr_confidence_score": "float (0.0–1.0) | null",
  "raw_source_path": "string (internal storage path)"
}
```

---

## INTELLIGENCE_ANALYSIS → HUMAN_REVIEW

### Input to INTELLIGENCE_ANALYSIS
Output of INGESTION (above).

### Output of INTELLIGENCE_ANALYSIS
```json
{
  "analysis_id": "string (UUID)",
  "doc_id": "string (ref)",
  "analysed_at": "datetime",
  "classification": [
    {
      "category": "Safety | Labeling | Manufacturing | Clinical",
      "confidence": "float (0.0–1.0)"
    }
  ],
  "impact_mapping": [
    {
      "entity_type": "Product | SOP | TherapeuticArea",
      "entity_id": "string",
      "entity_name": "string",
      "impact_rationale": "string",
      "affected": "boolean"
    }
  ],
  "risk_score": {
    "level": "High | Medium | Low",
    "score": "integer (1–100)",
    "rationale": "string",
    "factors": {
      "category_weight": "float",
      "affected_product_count": "integer",
      "regulatory_deadline_present": "boolean",
      "geographic_scope_count": "integer"
    }
  },
  "ai_summary": "string (max 300 words)",
  "model_version": "string"
}
```

---

## HUMAN_REVIEW → ORCHESTRATION or LEARNING_LOG

### Input to HUMAN_REVIEW
Output of INTELLIGENCE_ANALYSIS (above).

### Output of HUMAN_REVIEW
```json
{
  "review_id": "string (UUID)",
  "analysis_id": "string (ref)",
  "reviewed_at": "datetime",
  "reviewer": {
    "user_id": "string",
    "name": "string",
    "role": "Regulatory Affairs | QA | Medical Affairs"
  },
  "decision": "Approve | Modify | Reject",
  "rationale": "string | null",
  "approved_output": {
    "classification": ["array (same schema as analysis, may be modified)"],
    "impact_mapping": ["array (same schema as analysis, may be modified)"],
    "risk_score": {"object (same schema, may be modified)"},
    "final_summary": "string"
  },
  "original_ai_output_preserved": "boolean (always true)"
}
```

**Routing:**
- `decision: Approve | Modify` → **ORCHESTRATION**
- `decision: Reject` → **LEARNING_LOG**

---

## ORCHESTRATION → ACTION_TRACKING

### Input to ORCHESTRATION
Approved output from HUMAN_REVIEW.

### Output of ORCHESTRATION
```json
{
  "orchestration_id": "string (UUID)",
  "review_id": "string (ref)",
  "triggered_at": "datetime",
  "actions": [
    {
      "action_id": "string (UUID)",
      "action_type": "Alert | QMS_Draft | StakeholderNotification",
      "target": "string (system name, group name, or email address)",
      "status": "Sent | Failed | Pending",
      "timestamp": "datetime",
      "reference_id": "string (external system ID if available)"
    }
  ],
  "qms_draft_id": "string | null",
  "alert_channels_notified": ["Teams | Email"]
}
```

---

## ACTION_TRACKING → DISCOVERY (cycle reset)

### Input to ACTION_TRACKING
Output of ORCHESTRATION (above).

### Output of ACTION_TRACKING (logged metrics payload)
```json
{
  "tracking_id": "string (UUID)",
  "orchestration_id": "string (ref)",
  "cycle_completed_at": "datetime",
  "workflow_statuses": [
    {
      "action_id": "string (ref)",
      "final_status": "Completed | In_Progress | Failed",
      "resolved_at": "datetime | null"
    }
  ],
  "stakeholder_feedback": [
    {
      "user_id": "string",
      "alert_relevant": "boolean",
      "comments": "string | null"
    }
  ],
  "cycle_metrics": {
    "detection_to_review_hours": "float",
    "review_to_action_hours": "float",
    "false_positive_flagged": "boolean",
    "end_to_end_hours": "float"
  }
}
```

---

## LEARNING_LOG (terminal for rejected cycles)

### Input to LEARNING_LOG
HUMAN_REVIEW output where `decision: Reject`.

### Output of LEARNING_LOG (stored record)
```json
{
  "log_id": "string (UUID)",
  "review_id": "string (ref)",
  "doc_id": "string (ref)",
  "logged_at": "datetime",
  "rejection_rationale": "string",
  "ai_output_snapshot": {"object (full INTELLIGENCE_ANALYSIS output)"},
  "classification_category_rejected": ["string"],
  "flagged_for_retraining": "boolean"
}
```
