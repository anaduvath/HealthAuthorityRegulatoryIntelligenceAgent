# Use Case: Regulatory Intelligence Monitoring Agent

## Overview

| Field | Detail |
|---|---|
| **Use Case ID** | UC-001 |
| **Title** | Automated Health Authority Guidance Monitoring & Impact Assessment |
| **Version** | 1.0.0 |
| **Date** | 2026-03-28 |
| **Status** | Draft |

---

## Problem Statement

Pharmaceutical and medical device companies must continuously track regulatory guidance issued by Health Authorities (FDA, EMA, MHRA). This process is today largely manual — regulatory affairs teams periodically check authority websites, download documents, interpret impact, and route actions. This creates latency risk, inconsistent coverage, and significant human effort for low-value monitoring tasks.

---

## Goal

Deploy an autonomous AI agent that continuously monitors Health Authority sources, interprets new/updated guidance, assesses business impact against internal product portfolios and SOPs, and orchestrates downstream compliance actions — with a mandatory human-in-the-loop validation gate before any action is executed.

---

## Actors

| Actor | Type | Role |
|---|---|---|
| **Regulatory Affairs Specialist** | Human | Reviews and approves AI-generated interpretation |
| **QA Lead** | Human | Validates change control triggers |
| **Medical Affairs Team** | Human (Notified) | Receives stakeholder alerts |
| **Regulatory Intelligence Agent** | AI System | Executes all automated states |
| **QMS Platform** | External System | Receives change control drafts |
| **Alerting System** (Teams/Email) | External System | Delivers notifications |

---

## Scope

**In Scope:**
- Monitoring of FDA, EMA, and MHRA public guidance portals and RSS feeds
- Ingestion, OCR, and normalization of PDF and HTML regulatory documents
- AI-driven classification, impact mapping, and risk scoring
- Human review workflow with approve/reject/modify capability
- Downstream orchestration: alerts, QMS drafts, stakeholder notifications
- Feedback loop and metrics logging

**Out of Scope:**
- Submission management or eCTD authoring
- Internal SOP authoring
- Legal counsel workflows
- Monitoring of non-HA sources (journals, trade press)

---

## Primary Flow

1. Agent polls configured HA URLs and RSS feeds on a scheduled interval.
2. A metadata/checksum comparison detects a new or updated document.
3. The document is downloaded, OCR-processed, and normalized.
4. The AI classifies the document, maps it to affected products/SOPs, and assigns a risk score.
5. A human reviewer is presented with the AI summary alongside the source text.
6. Upon approval (or modification), orchestration tasks are triggered automatically.
7. Workflow statuses are monitored and feedback is logged before the cycle resets.

---

## Alternate Flows

| Scenario | Handling |
|---|---|
| Reviewer **rejects** AI interpretation | Log to LEARNING_LOG, reset to DISCOVERY without triggering any downstream action |
| Reviewer **modifies** AI interpretation | Modified output proceeds to ORCHESTRATION; original AI output is preserved for audit |
| OCR/ingestion failure | Retry up to 3 times; escalate to human if unresolved |
| QMS integration failure | Alert ops team; hold ORCHESTRATION state pending manual retry |
| No new documents detected | Remain in DISCOVERY; log poll timestamp |

---

## Success Criteria

| Metric | Target |
|---|---|
| Source polling latency | < 4 hours from HA publication |
| AI classification accuracy (post-review) | ≥ 90% approval rate without modification |
| Human review SLA | < 48 hours for High-risk; < 5 days for Medium/Low |
| False positive rate | < 15% over rolling 30-day window |
| QMS draft initiation time | < 1 hour post-approval |

---

## Assumptions & Constraints

- HA portals are publicly accessible without authentication.
- Internal product list and SOP registry are available via API or static file.
- QMS platform exposes an API for programmatic draft creation.
- All AI outputs are non-binding until human-approved.
- Data residency requirements comply with applicable regulations.
