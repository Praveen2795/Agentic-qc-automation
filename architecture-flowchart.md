# Agentic QC Automation — Architecture Flowchart

> End-to-End Architecture for Scalable Quality Control Automation — Worst-Case Design

---

## High-Level Overview

```mermaid
flowchart TB
    subgraph OVERVIEW["🏗️ HIGH-LEVEL ARCHITECTURE"]
        direction TB
        P0["🔬 Phase 0\nExperimentation & Validation"]
        PA["🔵 Phase A\nOnboarding (Done Once Per QC)"]
        PB["🟣 Phase B\nMonthly QC Execution"]
        INFRA["🟢 Shared Infrastructure\nReusable Building Blocks"]
        OUTPUT["🟠 Output\nHuman Review & Sign-off"]

        P0 -->|"Calibrates"| PA
        PA -->|"Produces locked config"| PB
        PB -->|"Generates reports"| OUTPUT
        INFRA -->|"Connectors + Rules + Correction Store"| PA
        INFRA -->|"Connectors + Rules"| PB
    end
```

---

## Phase 0 — Experimentation & Validation

```mermaid
flowchart LR
    subgraph PHASE0["🔬 PHASE 0 — Experimentation & Validation (Before Full Build)"]
        direction LR
        P0A["📄 Select 3-5\nPnP Documents"] --> P0B["🤖 Test LLM\nParsing"]
        P0B --> P0C["👤 SME Grades\nOutput"]
        P0C --> P0D["📊 Calibrate\nthe Design"]
    end

    subgraph OUTCOMES["Accuracy Outcomes"]
        direction LR
        O1["✅ 80%+\nAI-assisted primary\nReflection optional"]
        O2["⚠️ 50-80%\nMandatory Reflection\nFew-shot examples"]
        O3["🔴 Below 50%\nFine-tuning needed\nSection-by-section parsing"]
    end

    P0D --> O1
    P0D --> O2
    P0D --> O3
```

---

## Phase A — Onboarding a New QC Process (Done Once Per QC)

```mermaid
flowchart TB
    subgraph PHASE_A["🔵 PHASE A — Onboarding a New QC Process"]
        direction TB

        subgraph ROW1["Step 1: Extract & Map"]
            direction LR
            A1["📄 Policy Document\n(PnP)"]
            A2["🤖 AI Extracts Steps\n🏷️ CoT"]
            A3["🧩 AI Maps to Config\n🏷️ CoT + Few-Shot"]
            A1 --> A2 --> A3
        end

        subgraph ROW2["Step 2: Gap Analysis & SME Review"]
            direction LR
            A4["🔍 AI Gap Analysis\n─────────\nReasoning: gaps in this PnP\nMemory: predicted from past QCs\n─────────\n🏷️ CoT + Few-Shot"]
            A5["🤝 SME Collaborative Review\n─────────\n1. Answer AI's questions\n2. Confirm/reject predictions\n3. Add domain knowledge\n─────────\n🏷️ Human-in-the-Loop"]
            A4 --> A5
        end

        subgraph ROW3["Step 3: Reflect & Approve"]
            direction LR
            A6["🔄 AI Self-Review\n🏷️ Reflection · Max 3"]
            A7["✅ SME Final Approval"]
            A6 --> A7
        end

        ROW1 --> ROW2 --> ROW3

        A7 --> LOCK["🔒 Locked Config\n(versioned)"]

        subgraph IMPROVE["Continuous Improvement Loop"]
            direction LR
            C1["📈 SME Corrections\nCaptured"]
            C2["🧠 Correction\nStore"]
            C3["🎯 AI Gets Smarter\nfew-shot → RAG → fine-tune"]
            C1 --> C2 --> C3
        end

        A7 --> C1
        C3 -.->|"improves extraction"| A2
        C3 -.->|"predicts gaps"| A4

        subgraph NEWRULE["New Rule Path"]
            direction LR
            N1["⚠️ Unknown Check\nType Found"]
            N2["👨‍💻 Engineering\nBuilds Rule"]
            N3["♻️ Available to\nAll QCs"]
            N1 --> N2 --> N3
        end

        A3 -.->|"if rule type missing"| N1
    end
```

---

## Phase B — Monthly QC Execution

```mermaid
flowchart TB
    subgraph PHASE_B["🟣 PHASE B — Monthly QC Execution"]
        direction TB

        subgraph ROW1B["Step 1: Trigger → Orchestrate → Execute"]
            direction LR
            B1["⏰ Scheduled Trigger\n(monthly batch)"]
            B2["🤖 LLM Orchestrator\n─────────\nReads locked config\nReasons through execution\n🏷️ Tool Use + CoT"]
            B3["🔁 For Each Account\n─────────\n1. Call connectors\n2. Call rule functions\n3. PASS / FAIL / REVIEW"]
            B1 --> B2 --> B3
        end

        subgraph ROW2B["Step 2: Verify → Report"]
            direction LR
            B4["✅ Verification Layer\n─────────\nPython (no AI) audits:\nAll steps executed?\nCorrect tools called?\n🏷️ Deterministic"]
            B5["📊 Generate Report\n─────────\nExcel: PASS/FAIL/REVIEW\nwith evidence & trace"]
            B4 --> B5
        end

        ROW1B --> ROW2B
    end
```

---

## Shared Infrastructure — Reusable Building Blocks

```mermaid
flowchart TB
    subgraph INFRA["🟢 SHARED INFRASTRUCTURE"]
        direction TB

        subgraph CONNECTORS["System Connectors"]
            direction LR
            SC1["TSYS"] ~~~ SC2["Debt Manager"] ~~~ SC3["Imaging"] ~~~ SC4["+Others\n~15-20"]
        end

        subgraph RULES["Validation Rules"]
            direction LR
            R1["Compare\nNumbers"] ~~~ R2["Compare\nDates"] ~~~ R3["Thresholds"] ~~~ R4["Timelines"]
            R5["Doc Exists"] ~~~ R6["Conditional"] ~~~ R7["Note Analysis\n🏷️ LLM"]
        end

        subgraph MEMORY["Correction Store"]
            direction LR
            M1["🧠 Long-term memory\nSME corrections + gap-fills"]
            M2["few-shot → RAG → fine-tune"]
        end
    end
```

---

## Output — Human Review & Sign-off

```mermaid
flowchart TB
    subgraph OUTPUT["🟠 OUTPUT — Human Review & Sign-off"]
        direction TB
        O1["📋 Excel Report\nPASS / FAIL / REVIEW"]
        O2["🖥️ SME Review Dashboard\nREVIEW items first"]
        O3["✅ Human Final Approval"]
        O1 --> O2 --> O3
    end
```

---

## Three-Outcome Model (PASS / FAIL / REVIEW)

```mermaid
flowchart LR
    CHECK["Validation Check"] --> PASS["✅ PASS\nClear pass.\nDeterministic or\nhigh-confidence."]
    CHECK --> FAIL["❌ FAIL\nClear fail.\nDeterministic or\nhigh-confidence."]
    CHECK --> REVIEW["⚠️ REVIEW\nSystem couldn't determine.\nMissing data, edge case,\nor low LLM confidence.\nHuman must decide."]

    style PASS fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style FAIL fill:#ffebee,stroke:#e53935,stroke-width:2px
    style REVIEW fill:#fff3e0,stroke:#ff9800,stroke-width:2px
```

---

## Reasoning Techniques Used

```mermaid
flowchart LR
    subgraph TECHNIQUES["AI Reasoning Techniques"]
        direction TB
        T1["🔗 Chain of Thought (CoT)\nStep-by-step reasoning\nUsed: Extract, Map, Gap Analysis"]
        T3["🔄 Reflection\nAI self-reviews its output\nUsed: AI Self-Review (max 3)"]
        T4["🛠️ Tool Use\nAI selects and calls tools\nUsed: LLM Orchestrator (Phase B)"]
        T5["📝 Few-Shot Learning\nPast corrections as examples\nUsed: Map, Gap Analysis"]
        T6["👤 Human-in-the-Loop\nDomain knowledge + decisions\nUsed: SME Review, Final Approval"]
    end
```

---

## Worst-Case Safeguards

```mermaid
flowchart TB
    subgraph SAFEGUARDS["🔴 WORST-CASE SAFEGUARDS"]
        direction TB
        S1["Low AI Parsing Accuracy\n→ Reflection, few-shot, fine-tuning"]
        S2["Low Rule Coverage\n→ Add new rules anytime"]
        S3["Complex Source Systems\n→ Stage data in data lake"]
        S4["Accuracy Needs Improvement\n→ Incremental levers available"]
    end
```
