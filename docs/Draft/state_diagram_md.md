# State Diagram — Regulatory Intelligence Agent

## Visual Flow

```
                          ┌─────────┐
                          │  START  │
                          └────┬────┘
                               │
                               ▼
                    ┌──────────────────────┐
              ┌────▶│      DISCOVERY       │◀─────────────────────────┐
              │     │  Poll HA URLs & RSS  │◀──────────────────┐      │
              │     │  Compare checksums   │                   │      │
              │     └──────────┬───────────┘                   │      │
              │   No change    │  New / updated doc             │      │
              └────────────────┘                               │      │
                               │                               │      │
                               ▼                               │      │
                    ┌──────────────────────┐                   │      │
                    │      INGESTION       │──── Fail ─────────┘      │
                    │  Download PDF/HTML   │    (after retries)        │
                    │  OCR · normalize     │                           │
                    └──────────┬───────────┘                           │
                               │  Normalized                           │
                               ▼                                       │
                    ┌──────────────────────┐                           │
                    │  INTELLIGENCE_       │                           │
                    │     ANALYSIS         │                           │
                    │  Classify · Impact   │                           │
                    │  map · Risk score    │                           │
                    │  Generate summary    │                           │
                    └──────────┬───────────┘                           │
                               │  Always mandatory                     │
                               ▼                                       │
              ╔══════════════════════════════╗                         │
              ║    ⚠  HUMAN_REVIEW           ║                         │
              ║   [CRITICAL BOUNDARY]        ║                         │
              ║  AI summary vs source text   ║                         │
              ║  Approve · Modify · Reject   ║                         │
              ╚══════════╤═══════════╤═══════╝                         │
                         │           │                                 │
               Approve / │           │ Reject                          │
                 Modify  │           │                                 │
                         ▼           ▼                                 │
          ┌──────────────────┐  ┌──────────────────┐                   │
          │  ORCHESTRATION   │  │  LEARNING_LOG    │                   │
          │  Teams/Email     │  │  Store rejected  │───────────────────┘
          │  alerts · QMS    │  │  output · flag   │  Reset → DISCOVERY
          │  draft · notify  │  │  for improvement │
          └────────┬─────────┘  └──────────────────┘
                   │  Tasks dispatched
                   ▼
          ┌──────────────────┐
          │  ACTION_TRACKING │
          │  Monitor QMS     │
          │  workflows       │
          │  Collect         │
          │  feedback · log  │
          │  metrics         │
          └────────┬─────────┘
                   │  Metrics logged
                   └───────────────────────────────────────────────────┘
                                                          Reset → DISCOVERY
```

---

## State Transitions Table

| From State | To State | Condition |
|---|---|---|
| START | DISCOVERY | System initialisation |
| DISCOVERY | DISCOVERY | No new or updated document detected |
| DISCOVERY | INGESTION | New or updated document detected (checksum mismatch) |
| INGESTION | INTELLIGENCE_ANALYSIS | Document successfully normalised |
| INGESTION | DISCOVERY | Ingestion failed after 3 retries |
| INTELLIGENCE_ANALYSIS | HUMAN_REVIEW | Analysis complete — always mandatory, no bypass |
| HUMAN_REVIEW | ORCHESTRATION | Reviewer decision: Approve or Modify |
| HUMAN_REVIEW | LEARNING_LOG | Reviewer decision: Reject |
| ORCHESTRATION | ACTION_TRACKING | All downstream tasks dispatched |
| ACTION_TRACKING | DISCOVERY | Metrics logged — cycle complete |
| LEARNING_LOG | DISCOVERY | Pipeline reset — no downstream actions triggered |

---

## State Summary

| State | Type | Responsibility |
|---|---|---|
| DISCOVERY | Automated | Poll HA sources; detect new or changed documents |
| INGESTION | Automated | Download, OCR, and normalise document content |
| INTELLIGENCE_ANALYSIS | AI — Cognitive | Classify, impact-map, risk-score, summarise |
| HUMAN_REVIEW | **Human Gate** | Validate AI output; Approve, Modify, or Reject |
| ORCHESTRATION | Automated | Dispatch alerts, QMS drafts, stakeholder notifications |
| ACTION_TRACKING | Automated | Monitor workflows; collect feedback; log metrics |
| LEARNING_LOG | Automated | Store rejected outputs; flag for model improvement |

---

## Key Design Rules

- **HUMAN_REVIEW has no bypass path.** Every analysis must receive an explicit human decision before any downstream action is taken.
- **Rejection never triggers actions.** A Reject decision routes directly to LEARNING_LOG and resets to DISCOVERY. No alerts, QMS drafts, or notifications are produced.
- **Ingestion failure resets silently.** After 3 retries the pipeline returns to DISCOVERY and logs the error; it does not escalate to HUMAN_REVIEW.
- **The cycle is continuous.** Both the happy path (ACTION_TRACKING) and the rejection path (LEARNING_LOG) ultimately return to DISCOVERY, keeping the agent perpetually monitoring.
- **Original AI output is always preserved.** Even on Modify decisions, the unedited AI analysis is retained alongside the human-approved version for audit purposes.
