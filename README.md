# Agentic QC Automation

> End-to-End Architecture for Scalable Quality Control Automation at a Financial Institution

## Problem Statement

A large financial institution performs **300+ Quality Control (QC) processes** across business units (credit cards, collections, fraud, recovery). Each QC follows a Policy & Procedure (PnP) document that defines what to check, which systems to query, and how to determine pass/fail.

Today, QC analysts **manually** execute these checks — pulling data from multiple source systems, comparing values, verifying timelines, and producing Excel reports. This is:

- **Slow**: Each QC takes significant analyst time, every month
- **Error-prone**: Manual cross-referencing across systems
- **Not scalable**: Adding new QCs requires training analysts on new procedures
- **Inconsistent**: Different analysts may interpret the same PnP differently

## Solution Overview

An **agentic AI system** that automates QC execution through two phases:

| Phase | Purpose | Frequency | Intelligence |
|-------|---------|-----------|-------------|
| **Phase 0** | Experimentation & validation | Once (before build) | LLM + human evaluation |
| **Phase A** | Onboard a QC process | Once per QC (or when PnP changes) | Agentic AI (Planning, Reflection, Tool Use, CoT, ToT) |
| **Phase B** | Execute monthly QC checks | Monthly (scheduled batch) | LLM orchestrator + deterministic verification |

**Key insight**: Phase A is where the agentic intelligence lives — the AI reads policy documents, maps validation steps to rules, identifies gaps, self-reviews, and learns from corrections. Phase B is execution, where an LLM orchestrator calls the right tools in the right order, with a deterministic verification layer ensuring completeness and correctness.

## Architecture

The full architecture is available in two formats:

- **[qc-architecture.html](qc-architecture.html)** — Interactive HTML diagram (open in browser for best experience)
- **[architecture-flowchart.md](architecture-flowchart.md)** — Mermaid flowcharts (renders natively on GitHub)

---

## Phase 0 — Experimentation & Validation

**Purpose**: Validate LLM capabilities against real PnP documents before committing to the full build.

**Process**:
1. Select 3-5 representative PnP documents (varying complexity, business unit, format)
2. Test LLM parsing — can it identify validation steps, source systems, fields, and comparison types?
3. SMEs grade the output — what percentage of steps were correctly parsed?
4. Calibrate the design based on accuracy results

**Accuracy Outcomes**:
| Accuracy | Implication |
|----------|------------|
| **80%+** | AI-assisted path is primary. Reflection optional. SME review is quick validation. |
| **50-80%** | Mandatory reflection loop. SME review is significant editing. Add few-shot examples. |
| **Below 50%** | Add fine-tuning, structured prompting, section-by-section parsing. Heavy SME involvement. |

---

## Phase A — Onboarding a New QC Process

**Purpose**: Convert a PnP document into a machine-executable configuration file.  
**Frequency**: Done once per QC. Re-triggered only if the PnP changes.

### Step-by-Step Flow

#### 1. AI Extracts Steps `[CoT Prompting]`
- LLM reads the PnP document
- Extracts each validation step in natural language
- Uses relevant past corrections from the Correction Store as few-shot examples
- Example output: *"Verify the charge-off amount matches the TSYS balance within $0.01"*

#### 2. AI Maps to Config `[CoT Prompting] [Few-Shot]`
- For each extracted step, the AI selects:
  - **Rule type** (e.g., `cross_match_numeric`, `timeline_check`)
  - **Data source** (e.g., TSYS, Debt Manager)
  - **Specific fields** to compare
  - **Parameters** (tolerance, date range, etc.)
- Past corrections from the Correction Store guide the mapping
- Produces a complete **draft config file**

#### 3. AI Gap Analysis `[CoT Prompting] [Few-Shot from Correction Store]`
The Gap Analysis agent finds gaps using **two distinct strategies**:

**Category 1 — Detected in This PnP (CoT reasoning)**
- Reviews the draft config and catches **structural & logical gaps**:
  - Missing data source, field name, or tolerance
  - Implicit dependencies between steps
  - Ambiguous ordering or edge cases

**Category 2 — Predicted from Similar QCs (Correction Store retrieval)**
- Retrieves past completed QC configs from the **Correction Store** — focusing on QCs with similar business units, rule types, or source systems
- Surfaces patterns the AI learned from previous onboardings:
  - *"3 previous QCs using TSYS + Debt Manager required a Thursday-only comparison rule — does this apply here?"*
  - *"Collections QCs in this BU always needed a balance threshold exception under $500"*
  - *"Last 2 QCs in this business unit referenced Legacy Table X for migrated accounts"*
- These are presented as **"Predicted gaps based on similar QCs"** — clearly separated from detected gaps — so the SME can confirm or reject each one

**Why this matters**: Early on (QC #1-5), the AI only catches ~30-40% of gaps (structural/logical). But as the Correction Store grows, the AI starts **anticipating domain-specific gaps** it learned from past SME sessions. By QC #50+, gap detection improves dramatically because the system has seen patterns across dozens of onboardings.

- Generates a **PnP Quality Score** — gives the organization data on which PnPs need rewriting

#### 4. SME Collaborative Review `[Human-in-the-Loop] [Domain Knowledge Capture]`
This is the most critical step in onboarding. The SME does three things:

**Part 1 — Answer AI's Questions (Detected Gaps)**
- Resolve the gaps AI identified in this PnP: missing tolerances, data sources, dependencies
- If a step has multiple possible meanings, AI presents options for SME to choose from

**Part 2 — Confirm/Reject AI Predictions (Predicted Gaps)**
- Review gaps the AI predicted based on similar past QCs from the Correction Store
- Confirm ones that apply, reject ones that don't — both responses further train the system

**Part 3 — Add Domain Knowledge (AI couldn't know to ask)**
- System migration exceptions: *"Migrated accounts use Legacy Table X, not TSYS"*
- Regulatory changes: *"RPC definition changed after 2023 regulatory update — PnP hasn't been updated"*
- Data timing constraints: *"TSYS updates daily but Debt Manager updates weekly — only compare after Thursday"*
- Business rule overrides: *"Skip this check for accounts under $500"*
- Unwritten workarounds: *"For accounts opened before 2019, use System B instead of System A"*

This is where the **real domain intelligence** enters the system. The config becomes smarter than the PnP — it captures knowledge that no document contains. All domain knowledge captured here is stored in the Correction Store for future QC onboardings.

#### 5. AI Self-Review (Reflection) `[Reflection Pattern] [Max 3 Iterations]`
- Reviews the **gap-filled** config
- Checks for missed steps, incorrect rule mappings, or inconsistencies introduced during gap-filling
- Runs up to **3 iterations maximum**
- If still uncertain after 3 loops, flags items for SME

#### 6. SME Final Approval
- QC analyst reviews the complete config — including gap-fills and AI self-review results
- Approves, adjusts, or overrides
- **Nothing runs without human sign-off**

#### 7. Locked & Versioned Configuration
- The approved config is saved as the **single source of truth**
- Specifies exactly which systems to query, which fields to compare, and which validation rules to use
- **Versioned for audit trail**

### Continuous Improvement Loop

When the SME adjusts the AI output, the system captures:

| Artifact | Description |
|----------|------------|
| **PnP text snippet** | What the AI was given (input) |
| **AI's original output** | What it generated (possibly wrong) |
| **SME's corrected version** | The ground truth |
| **Gap-fills** | Knowledge not in the PnP that the SME provided |
| **Metadata** | QC name, business unit, rule type, date |

These are stored in the **Correction Store** (long-term memory). The next QC onboarding pulls relevant past corrections as few-shot examples into **three** downstream steps:

| Feed-Forward Target | What It Gets | Impact |
|---|---|---|
| **AI Extracts Steps** | Past corrections improve parsing accuracy | Fewer extraction errors over time |
| **AI Gap Analysis** | Past gap-fills predict gaps in new QCs | AI anticipates domain-specific gaps |
| **Phase B Execution** | Past patterns improve orchestration | Better tool selection and edge case handling |

**Scaling path**:
- **Early (QC #1-10)**: Few-shot — attach 2-3 similar completed configs as examples
- **Growth (QC #10-50)**: RAG — semantic search over all past gaps by business unit, rule type, source systems
- **Scale (QC #50+)**: The model has seen enough patterns that gap detection becomes genuinely predictive
- **Late (QC #200+)**: Fine-tuning if needed

### New Rule Type Discovery

If the AI encounters a validation step that doesn't match any existing rule type:
1. It flags the step and routes it to the engineering team
2. Engineering builds a new reusable rule function (standard dev → QA → prod)
3. The new rule becomes available to **all** QCs — not just this one
4. This need **diminishes over time** as the rule library grows

---

## Phase B — Monthly QC Execution

**Purpose**: Execute all onboarded QCs automatically each month.  
**Frequency**: Monthly scheduled batch job (with manual re-run option).

### Step-by-Step Flow

#### 1. Scheduled Batch Trigger
- Runs automatically on a fixed monthly schedule (e.g., 3rd business day)
- All QCs execute overnight
- Manual re-run available on-demand for exceptions

#### 2. LLM Orchestrator `[Tool Use Pattern] [CoT Prompting]`
- Reads the **locked config** file
- **Reasons** through execution — decides the order to call tools, handles step dependencies, manages multi-hop lookups, and applies fallback logic
- This is an LLM (not a static Python script) because:
  - QCs have complex **dependencies** between steps
  - Some require **multi-hop lookups** (account → case → document)
  - **Novel patterns** across 300+ QCs may need flexible handling
  - **Edge cases** are flagged as REVIEW rather than crashing

#### 3. For Each Account in Sample
- **Step 1**: LLM calls the right connectors to pull data (handles multi-hop)
- **Step 2**: LLM calls the right rule functions for each validation check, passing the correct data
- **Step 3**: Each check returns **PASS**, **FAIL**, or **REVIEW** with evidence

#### 4. Deterministic Verification Layer `[Deterministic Python] [Audit Guard]`
- A **Python script (no AI)** that audits every LLM-orchestrated run:
  - Did the LLM execute **every** step listed in the config? *(completeness)*
  - Did it call the **correct tools** for each step? *(correctness)*
  - Did it use the **correct data sources**? *(accuracy)*
  - Were any steps **skipped or invented**? *(hallucination guard)*
- If verification fails → entire QC run is flagged as **REVIEW** with the reason

#### 5. Generate Report
- Results compiled into a structured Excel report (same format teams use today)
- Each check marked **PASS**, **FAIL**, or **REVIEW**
- Includes full execution trace and evidence

---

## Three-Outcome Model

Every validation check produces one of three results:

| Result | Meaning | Action Required |
|--------|---------|-----------------|
| **PASS** | Clear pass. Deterministic or high-confidence. | Spot-check only |
| **FAIL** | Clear fail. Deterministic or high-confidence. | Review evidence |
| **REVIEW** | System couldn't determine. Missing data, edge case, or low LLM confidence. | **Human must decide** |

REVIEW is the safety net. When the system is uncertain, it doesn't guess — it asks a human. This applies to:
- Missing or malformed data
- Edge cases in deterministic rules
- Low-confidence LLM note analysis
- Verification layer failures (LLM skipped a step, used wrong tool, etc.)

---

## Shared Infrastructure

### System Connectors
Built once per source system, reused across all QCs:

| Connector | System | Purpose |
|-----------|--------|---------|
| TSYS | Card processing | Account balances, transaction data |
| Debt Manager | Collections | Charge-off data, payment history |
| Imaging | Document management | Letters, notices, scanned documents |
| + Others | Various | ~15-20 total estimated |

### Validation Rules (Reusable Library)
Pre-built functions, each handling one type of check:

| Rule Type | Description | Engine |
|-----------|-------------|--------|
| `cross_match_numeric` | Compare dollar amounts, balances | Deterministic Python |
| `cross_match_date` | Compare dates, deadlines | Deterministic Python |
| `cross_match_string` | Compare text values | Deterministic Python |
| `threshold_check` | Value above/below a limit | Deterministic Python |
| `timeline_check` | Action within N days | Deterministic Python |
| `document_exists` | Letter/notice on file | Deterministic Python |
| `conditional_check` | If X, then Y must be true | Deterministic Python |
| `list_contains` | Value must be in allowed set | Deterministic Python |
| `note_analysis` | Evaluate free-text agent notes | **LLM** with structured output |

Estimate: ~8-10 rule types will cover most QCs. New types can be added at any time. The `note_analysis` rule uses LLM because free-text agent notes cannot be parsed deterministically — it outputs structured results with a REVIEW flag when confidence is low.

### Correction Store (AI Memory)
Persistent long-term memory that stores:
- SME corrections from every onboarding
- Gap-fills (knowledge not in the PnP)
- Common defaults learned over time
- **Confirmed/rejected gap predictions** (further refines future predictions)

Fed back into **three** steps: Extract, Gap Analysis, and Phase B Execution. The Gap Analysis feed-forward is the most impactful — it transforms the agent from "find what's logically missing in THIS PnP" → "predict what's USUALLY missing in PnPs LIKE this, based on every QC we've ever onboarded."

Scales from few-shot → RAG retrieval → fine-tuning as the library grows.

---

## Reasoning Techniques

| Technique | What It Does | Where Used |
|-----------|-------------|------------|
| **Chain of Thought (CoT)** | Forces step-by-step reasoning in prompts | AI Extracts Steps, AI Maps to Config, AI Gap Analysis, LLM Orchestrator |
| **Human-in-the-Loop** | Human provides domain knowledge, confirms/rejects AI predictions, resolves ambiguity | SME Collaborative Review (Phase A), SME Final Approval, Human Final Approval (Output) |
| **Reflection** | AI self-reviews its own output, catches errors | AI Self-Review (max 3 iterations) |
| **Tool Use** | AI selects and calls appropriate tools at runtime | LLM Orchestrator (Phase B — calls connectors and rules) |
| **Few-Shot Learning** | Past corrections as in-context examples | AI Maps to Config, **AI Gap Analysis** (via Correction Store) |

---

## Design Principles

| Principle | Description |
|-----------|------------|
| **Configuration-Driven** | Adding a new QC = adding a new config file, not writing new code. Target: 30-40 QCs onboarded per quarter. |
| **Deterministic + Auditable** | Most rules are deterministic Python. LLM orchestration is audited by a verification layer. Note analysis uses LLM with structured output and REVIEW flags. Full audit trail for every execution. |
| **Diminishing Engineering Effort** | Rule types and connectors are reusable. By Q3-Q4, most new QCs require zero code changes — just configuration. The Correction Store makes onboarding faster over time. |
| **Human-in-the-Loop** | Humans approve all configs (Phase A) and sign off on all results (Output). No automated production actions. No QC is ever closed automatically. |

---

## Worst-Case Safeguards

| Scenario | Safeguard |
|----------|-----------|
| **AI parsing accuracy is low** | Reflection loop, few-shot examples from SME corrections, structured prompting, section-by-section parsing. Accuracy improves with every QC onboarded. |
| **Rule coverage is low** | Architecture supports adding new rule types at any time. Each new rule immediately becomes available to all QCs. |
| **Source systems are complex** | Data can be staged into a data lake first. Agents query staged data instead of live systems — safer and more auditable. |
| **LLM orchestrator deviates** | Deterministic verification layer audits every run. If any step is skipped, wrong tool called, or data source misused → entire QC flagged as REVIEW. |
| **PnP documents have gaps** | AI Gap Analysis identifies missing/ambiguous items. SME fills gaps with tribal knowledge. Both corrections and gap-fills stored in Correction Store for future use. |

---

## Design Assumptions (Worst-Case Planning)

This architecture is designed for the worst-case scenario. We have not yet validated LLM performance against real PnP documents.

1. **LLM parsing accuracy is unknown** — could be as low as 50% initially. Design includes self-improvement loop.
2. **PnP documents vary widely** — structured, narrative, and scanned formats. AI pipeline handles multiple formats with reflection.
3. **Pass/fail logic is mostly deterministic** — exception: note analysis uses LLM with REVIEW flag. Humans make the final call.
4. **No automated production actions** — system generates reports for human review. Does not close QCs or update records.
5. **~8-10 rule types will cover most QCs** — estimate to be refined during experimentation. Architecture supports new types anytime.
6. **Experimentation phase is required** — Phase 0 validates LLM accuracy, rule coverage, and connector feasibility with real data.

---

## Technology Stack (Recommended)

| Component | Technology |
|-----------|-----------|
| LLM Provider | AWS Bedrock (Claude / GPT-4) |
| Orchestration | Python + LangChain / LangGraph |
| Correction Store | PostgreSQL (pgvector for RAG) or DynamoDB |
| Config Format | YAML / JSON (versioned in Git) |
| Connectors | Python adapters per source system |
| Rules Engine | Python functions (deterministic) |
| Reporting | openpyxl (Excel generation) |
| Scheduling | AWS EventBridge / Step Functions |
| Dashboard | Streamlit or internal web app |

---

## Getting Started

### For Contributors
1. Review the [architecture flowchart](architecture-flowchart.md) for visual understanding
2. Open [qc-architecture.html](qc-architecture.html) in a browser for the detailed interactive diagram
3. Start with Phase 0 — experiment with 3-5 real PnP documents

### For AI Agents Building on This
If you're an AI agent tasked with implementing this system, here's the build order:

1. **Phase 0**: Build a simple PnP parsing pipeline and test against real documents
2. **Connectors**: Build adapters for each source system (start with TSYS, Debt Manager)
3. **Rules**: Implement the ~8-10 rule functions as Python functions
4. **Phase A Pipeline**: Build the onboarding flow (extract → map → gap analysis → reflection → approval)
5. **Correction Store**: Set up the database for storing SME corrections
6. **Config Schema**: Define the YAML/JSON schema for QC configs
7. **Phase B Pipeline**: Build the LLM orchestrator + verification layer
8. **Reporting**: Build the Excel report generator
9. **Dashboard**: Build the SME review dashboard

Each component should be independently testable. The config schema is the contract between Phase A (producer) and Phase B (consumer).

---

## License

Internal use only. Confidential.
