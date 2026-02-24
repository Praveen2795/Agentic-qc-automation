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
            A1["📄 Policy Document\n(PnP)"] --> A2["🤖 AI Extracts Steps\n━━━━━━━━━━\nLLM reads PnP, extracts\nvalidation steps in\nnatural language\n━━━━━━━━━━\n🏷️ CoT Prompting"]
            A2 --> A3["🧩 AI Maps to Config\n━━━━━━━━━━\nSelects rule type,\ndata source, and fields\nfor each step.\nProduces draft config.\n━━━━━━━━━━\n🏷️ CoT + Few-Shot"]
        end

        subgraph ROW2["Step 2: Gap Analysis"]
            direction LR
            A4["🔍 AI Gap Analysis\n━━━━━━━━━━\nFlags missing/ambiguous items:\n• Missing tolerance/system\n• No fallback source\n• Implicit dependencies\n━━━━━━━━━━\nGenerates PnP Quality Score\n🏷️ CoT Prompting"] --> A5["👤 SME Fills Gaps\n━━━━━━━━━━\nProvides tribal knowledge,\nworkarounds, dependencies,\ntolerances, fallback sources.\nAI asks, SME answers.\n━━━━━━━━━━\n🏷️ ToT — Multiple Interpretations"]
        end

        subgraph ROW3["Step 3: Reflect & Approve"]
            direction LR
            A6["🔄 AI Self-Review\n(Reflection)\n━━━━━━━━━━\nReviews gap-filled config.\nChecks for missed steps,\nwrong mappings,\ninconsistencies.\n━━━━━━━━━━\n🏷️ Reflection Pattern\n🏷️ Max 3 Iterations"] --> A7["✅ SME Final Approval\n━━━━━━━━━━\nReviews complete config.\nApproves, adjusts, or\noverrides. Nothing runs\nwithout human sign-off."]
        end

        ROW1 --> ROW2
        ROW2 --> ROW3

        A7 --> LOCK["🔒 Locked & Versioned Configuration\nSingle source of truth.\nSpecifies systems, fields, and rules.\nVersioned for audit trail."]

        subgraph IMPROVE["Continuous Improvement Loop"]
            direction LR
            C1["📈 SME Corrections\nCaptured\n━━━━━━━━━━\nCaptures corrections\nand gap-fills as\nstructured records"] --> C2["🧠 Correction Store\n━━━━━━━━━━\nLong-term memory.\nPnP snippet → AI output\n→ SME-corrected output.\nStores common defaults."]
            C2 --> C3["🎯 AI Gets Smarter\nOver Time\n━━━━━━━━━━\nFew-shot → RAG →\nFine-tuning as\nlibrary grows"]
        end

        A7 --> C1
        C3 -.->|"Fed back into next onboarding"| A2

        subgraph NEWRULE["If Unknown Check Type Found"]
            direction LR
            N1["⚠️ New Check\nType Found"] --> N2["👨‍💻 Engineering\nBuilds New Rule"] --> N3["♻️ Available\nfor All QCs"]
        end

        A3 -.->|"If rule type doesn't exist"| N1
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
            B1["⏰ Scheduled\nBatch Trigger\n━━━━━━━━━━\nFixed monthly schedule\n(e.g., 3rd business day)\nManual re-run available"] --> B2["🤖 LLM Orchestrator\n━━━━━━━━━━\nReads locked config.\nReasons through execution:\n• Tool call order\n• Step dependencies\n• Multi-hop lookups\n• Fallback logic\n━━━━━━━━━━\n🏷️ Tool Use Pattern\n🏷️ CoT Prompting"]
            B2 --> B3["🔁 For Each Account\nin Sample\n━━━━━━━━━━\n1. LLM calls connectors\n   (multi-hop: acct→case→doc)\n2. LLM calls rule functions\n3. Returns PASS / FAIL / REVIEW\n   with evidence"]
        end

        subgraph ROW2B["Step 2: Verify → Report"]
            direction LR
            B4["✅ Deterministic\nVerification Layer\n━━━━━━━━━━\nPython script (no AI) audits:\n• All config steps executed?\n• Correct tools called?\n• Correct data sources used?\n• Any steps skipped/invented?\n━━━━━━━━━━\nIf verification fails →\nentire QC = REVIEW\n━━━━━━━━━━\n🏷️ Deterministic Python\n🏷️ Audit Guard"] --> B5["📊 Generate Report\n━━━━━━━━━━\nStructured Excel report.\nEach check: PASS / FAIL / REVIEW\nFull execution trace.\nSame format teams use today."]
        end

        ROW1B --> ROW2B
    end
```

---

## Shared Infrastructure — Reusable Building Blocks

```mermaid
flowchart TB
    subgraph INFRA["🟢 SHARED INFRASTRUCTURE — Reusable Building Blocks"]
        direction TB

        subgraph CONNECTORS["System Connectors (Built Once Per System)"]
            direction LR
            SC1["TSYS\nCard processing"]
            SC2["Debt Manager\nCollections"]
            SC3["Imaging\nDocuments"]
            SC4["+ Others\n~15-20 estimated"]
        end

        subgraph RULES["Validation Rules (Reusable Library)"]
            direction LR
            R1["Compare Numbers\nDollar amounts"]
            R2["Compare Dates\nDeadlines"]
            R3["Check Thresholds\nAbove/below limits"]
            R4["Verify Timelines\nAction within N days"]
            R5["Document Exists\nLetter on file?"]
            R6["Conditional Logic\nIf X, then Y"]
            R7["Note Analysis 🏷️LLM\nFree-text agent notes"]
        end

        subgraph MEMORY["AI Memory (Correction Store)"]
            direction LR
            M1["🧠 Correction Store\nStores SME corrections\nas structured records.\nFed back as few-shot\nexamples at onboarding."]
            M2["Scaling path:\nFew-shot → RAG → Fine-tuning"]
        end
    end
```

---

## Output — Human Review & Sign-off

```mermaid
flowchart TB
    subgraph OUTPUT["🟠 OUTPUT — Human Review & Sign-off"]
        direction TB
        O1["📋 Structured Excel Report\nPASS / FAIL / REVIEW with evidence.\nREVIEW items surfaced first."]
        O1 --> O2["🖥️ SME Review Dashboard\nAnalysts focus on REVIEW items.\nPASS/FAIL spot-checked for quality.\nEach REVIEW resolved as pass or fail."]
        O2 --> O3["✅ Human Final Approval\nHuman SME has final sign-off.\nSystem assists — does not decide.\nAutomation of execution, not autonomy."]
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
        T1["🔗 Chain of Thought (CoT)\nStep-by-step reasoning in prompts.\nUsed in: Extract, Map, Gap Analysis"]
        T2["🌳 Tree of Thought (ToT)\nExplore multiple interpretations.\nUsed in: SME Fills Gaps\n(when PnP is ambiguous)"]
        T3["🔄 Reflection\nAI self-reviews its own output.\nUsed in: AI Self-Review\n(max 3 iterations)"]
        T4["🛠️ Tool Use\nAI selects and calls tools.\nUsed in: LLM Orchestrator\n(Phase B execution)"]
        T5["📝 Few-Shot Learning\nPast corrections as examples.\nUsed in: AI Maps to Config\n(via Correction Store)"]
    end
```

---

## Worst-Case Safeguards

```mermaid
flowchart TB
    subgraph SAFEGUARDS["🔴 WORST-CASE SAFEGUARDS"]
        direction TB
        S1["If AI Parsing Accuracy is Low\n→ Reflection loop, few-shot examples,\nstructured prompting, section-by-section parsing"]
        S2["If Rule Coverage is Low\n→ Add new rule types anytime.\nEach new rule available to all QCs."]
        S3["If Source Systems Are Complex\n→ Stage data into a data lake.\nAgents query staged data."]
        S4["If Accuracy Needs Improve\n→ Reflection, few-shot, fine-tuning.\nEach added incrementally."]
    end
```
