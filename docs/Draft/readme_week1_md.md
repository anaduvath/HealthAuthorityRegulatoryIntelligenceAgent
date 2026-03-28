# Week 1 — Regulatory Intelligence Agent: Project Kickoff

## Project Summary

This project delivers an AI-powered autonomous agent that monitors Health Authority (FDA, EMA, MHRA) regulatory guidance, interprets its impact on internal product portfolios and SOPs, and orchestrates downstream compliance workflows — with a mandatory human validation gate before any action is taken.

---

## Repository Structure

```
regulatory-intelligence-agent/
│
├── docs/
│   ├── use_case.md               # Business problem, actors, scope, success criteria
│   ├── agent_states.md           # Full state-by-state specification
│   ├── agent_io_contract.md      # Input/output JSON schemas per state transition
│   ├── state_diagram.mmd         # Mermaid state diagram (render with Mermaid Live or VS Code)
│   ├── data_spec.md              # All data stores, schemas, and retention policies
│   └── README_week1.md           # This file
│
├── src/                          # (To be populated in Week 2+)
│   ├── discovery/
│   ├── ingestion/
│   ├── intelligence_analysis/
│   ├── human_review/
│   ├── orchestration/
│   └── action_tracking/
│
├── tests/                        # (To be populated in Week 2+)
├── config/
│   └── sources.yaml              # HA polling configuration (URLs, intervals)
└── .env.example                  # Environment variable template
```

---

## Week 1 Deliverables — Status

| Deliverable | File | Status |
|---|---|---|
| Use Case Document | `docs/use_case.md` | ✅ Complete |
| Agent States Specification | `docs/agent_states.md` | ✅ Complete |
| Agent I/O Contract | `docs/agent_io_contract.md` | ✅ Complete |
| State Diagram | `docs/state_diagram.mmd` | ✅ Complete |
| Data Specification | `docs/data_spec.md` | ✅ Complete |
| README (this file) | `docs/README_week1.md` | ✅ Complete |

---

## Architecture at a Glance

```
  ┌─────────────────────────────────────────────────────────┐
  │                    Agent Pipeline                        │
  │                                                          │
  │  DISCOVERY → INGESTION → INTELLIGENCE_ANALYSIS          │
  │                                    │                     │
  │                           ┌────────▼────────┐           │
  │                           │  HUMAN_REVIEW   │ ← Gate    │
  │                           └───┬─────────┬───┘           │
  │                    Approve/   │         │  Reject        │
  │                    Modify     ▼         ▼                │
  │               ORCHESTRATION   LEARNING_LOG               │
  │                    │                   │                 │
  │                    ▼                   │                 │
  │             ACTION_TRACKING            │                 │
  │                    │                   │                 │
  │                    └──────────────────►│                 │
  │                        Reset to DISCOVERY                │
  └─────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions

### 1. Mandatory Human Gate
The `HUMAN_REVIEW` state has no bypass path. Any approved or modified output is recorded alongside the preserved original AI output for full auditability.

### 2. Rejection Does Not Trigger Actions
When a reviewer rejects AI output, the pipeline routes to `LEARNING_LOG` and resets to `DISCOVERY`. No alerts, QMS drafts, or stakeholder notifications are ever triggered for rejected analyses.

### 3. Immutable Audit Logs
The `Review_Log` is append-only. No update or delete operations are permitted on human decisions. This satisfies regulatory audit trail requirements.

### 4. Modular State Design
Each state is independently deployable as a microservice or serverless function. State transitions are event-driven (message queue), allowing each component to be scaled, tested, and maintained in isolation.

### 5. Separation of Reference Data
`Product_Registry` and `SOP_Registry` are managed externally by Regulatory Affairs / MDM teams and consumed as read-only references by the agent. This prevents tight coupling between the agent and internal master data systems.

---

## Glossary

| Term | Definition |
|---|---|
| **HA** | Health Authority (FDA, EMA, MHRA) |
| **QMS** | Quality Management System |
| **SOP** | Standard Operating Procedure |
| **OCR** | Optical Character Recognition |
| **Impact Mapping** | Cross-referencing guidance content against internal product/SOP registries |
| **Risk Score** | AI-assigned severity level (High/Medium/Low) based on classification and scope |
| **Change Control** | Formal QMS process to document and approve changes to regulated processes |
| **Known_Guidance_DB** | Internal registry of previously seen HA documents with checksums |
| **Learning_Log** | Storage of rejected AI outputs used for model improvement |

---

## Week 2 Planned Work

- [ ] Implement DISCOVERY poller (FDA RSS feed + EMA portal)
- [ ] Stand up `Known_Guidance_DB` schema and migrations
- [ ] Build INGESTION pipeline with PDF OCR (PyMuPDF / Tesseract)
- [ ] Define prompting strategy for INTELLIGENCE_ANALYSIS (LLM selection + prompt templates)
- [ ] Prototype HUMAN_REVIEW UI wireframes (React or internal portal)
- [ ] Set up CI/CD pipeline and repository scaffolding

---

## How to Render the State Diagram

The file `state_diagram.mmd` is a Mermaid diagram. To render it:

- **VS Code:** Install the [Mermaid Preview](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid) extension.
- **Online:** Paste contents into [mermaid.live](https://mermaid.live).
- **GitHub/GitLab:** Mermaid diagrams render natively in Markdown files when wrapped in a `mermaid` code block.

---

## Questions & Contact

Raise questions as issues in the project repository. Tag issues with `week1-docs` for documentation-related queries.
