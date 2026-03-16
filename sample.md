# GenAI XGV Template Engine

> *"From 5,000 lines of code per report to zero — let AI write the template, you define the rules."*

**Intelligent. Scalable. Report-Agnostic.**

---

## 📖 Table of Contents

- [What is This Project?](#what-is-this-project)
- [Background — Regulatory Reporting at Citi](#background--regulatory-reporting-at-citi)
- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [How It Works — Technical Flow](#how-it-works--technical-flow)
- [Architecture — 4 Layers](#architecture--4-layers)
- [Inputs Explained](#inputs-explained)
- [Why LLM? Why Not Just Python/Java?](#why-llm-why-not-just-pythonjava)
- [Validated Results](#validated-results)
- [Impact](#impact)

---

## What is This Project?

The **GenAI XGV Template Engine** is an AI-powered system that automatically generates XBRL regulatory report templates for Citi's regulatory submissions — eliminating the need for developers to hand-code them.

You give it **2 files**. It gives you back a **production-ready XML template**. No Java code written. No report-specific logic. No weeks of waiting.

---

## Background — Regulatory Reporting at Citi

Banks like Citi are legally required to submit reports to government regulators periodically. For example:

- 🇮🇳 **India** → Reserve Bank of India (RBI)
- 🇸🇬 **Singapore** → Monetary Authority of Singapore (MAS)
- 🇭🇰 **Hong Kong** → HKMA
- 🇦🇺 **Australia** → APRA
- 🇹🇭 **Thailand** → BOT

These reports contain details like loans, NPAs, interest income, and more — filed in a strict machine-readable format called **XBRL (eXtensible Business Reporting Language)**.

### What is XBRL?

XBRL is the standardized XML format regulators demand. Every value in the report (called a **"fact"**) must be tagged with the exact field name, data type, unit, and category defined by the regulator's **taxonomy**.

```xml
<rbi:LoansPrioritySector
    contextRef="Domestic_Q1_2026"
    unitRef="INR"
    decimals="2">
    5000000.00
</rbi:LoansPrioritySector>
```

### What is a Taxonomy?

The regulator publishes a **taxonomy** — a dictionary + rulebook defining every field, its data type, units, dimensions, and allowed values. Think of it as a **database schema published by the government** that every bank must conform to.

### What is the Excel Spec?

The regulator also provides an **iFile Excel template** — a visual representation of the report with hidden cells containing machine-readable mappings (concept refs, dimension refs, table markers).

---

## The Problem

Every time Citi needed to onboard a new regulatory report, a developer had to:

| Pain Point | Detail |
|---|---|
| **Write report-specific Java code** | 5,010 lines across 4 generators per report |
| **Time per report** | 2–4 weeks of specialized developer time |
| **Bus factor** | Only 2–3 developers understand the system |
| **Regression risk** | A fix in one report frequently breaks others |
| **Scalability** | 3 reports in 6 months; 100+ reports needed across 5 countries |

The root cause: every time the government updates their taxonomy (new fields, new formats, new rules), all the hardcoded rules in Java need to be manually updated. This is unsustainable.

### What Happens When Taxonomy Changes (Old Way)

```
Government updates taxonomy
        ↓
New fields, new formats, new rules
        ↓
Developer manually updates Java code
        ↓
2–4 weeks of work
        ↓
Regression risk — fixing one report breaks another
        ↓
Repeat for EVERY report, EVERY country
```

---

## The Solution

A **Gen AI-powered, LLM-agnostic engine** that transforms a Taxonomy ZIP and an Excel Spec into a production-ready XML template — without writing a single line of report-specific code.

### What Happens When Taxonomy Changes (New Way)

```
Government updates taxonomy
        ↓
New Taxonomy ZIP + Excel Spec downloaded
        ↓
Python extracts the updated JSONs (automatically)
        ↓
LLM reads the new JSONs and figures out
the correct mappings, data types, and conventions
        ↓
New template.xml generated
        ↓
Done. Zero code changes. Under 2 hours.
```

---

## How It Works — Technical Flow

```
Taxonomy ZIP + Excel Spec (iFile)
        ↓
  [Python Extractors]
  Parse files, extract structured data
        ↓
  taxonomy.json + excel_spec.json
        ↓
  [LLM — GPT-4 / Claude / Gemini / Devin]
  AI intelligently maps concepts, contexts,
  units, decimals, and pipe transformations
        ↓
  template_structure.json
        ↓
  [Deterministic JSON → XML Builder]
        ↓
  template.xml
  (contains placeholders like ${Sheet<Cell>})
        ↓
  [Existing Java XMLConverter — UNCHANGED]
  Plugs real Citi bank data into placeholders
        ↓
  Final XBRL XML ✅
  Submitted to regulator
```

> **Key principle:** The AI generates the *template*, not the final output. Real bank data only enters in the Java layer — ensuring control, compliance, and security.

---

## Architecture — 4 Layers

### Layer 1 — Input Sources
Only **2 user inputs** required:

| Input | Description |
|---|---|
| **Taxonomy ZIP** | Published by the regulator (RBI, MAS, etc.). Contains XSD/XML files defining all concepts, dimensions, and rules. |
| **Specification Excel (iFile)** | Also published by the regulator. Contains the visible report layout + hidden cells with machine-readable concept and dimension mappings. |

---

### Layer 2 — Intelligent Extraction (Python — 100% Deterministic, No AI)

Python reads both input files and converts them into clean, structured JSON:

**Taxonomy JSON Extractor:**
1. Unzips the taxonomy package
2. Parses `taxonomyPackage.xml` → finds the entry point for the target report
3. Parses XSD files → extracts concepts, data types, units, decimals
4. Parses table and definition linkbases → extracts table layouts and dimensions
5. Outputs → `taxonomy.json`

**Excel Spec JSON Extractor:**
1. Reads ALL sheets including hidden and veryHidden sheets
2. Parses Hidden Column A → concept references (row-axis)
3. Parses Hidden Column B → dimension::member references
4. Parses Hidden Column C → `#TABLE#` markers and layout IDs
5. Parses hidden rows → column-axis concept references
6. Parses veryHidden sheets (`+Lineitems`, `StartUp`, `Config`, `MainSheet`)
7. Cross-validates with `taxonomy.json`
8. Outputs → `excel_spec.json`

> **Note:** Cell comments are NOT used. All data comes from hidden cells already embedded by the RBI iFile tool.

---

### Layer 3 — Gen AI Template Engine (Python — LLM-Powered, ONLY AI Layer)

The LLM receives both JSONs via the **P003 prompt** and makes intelligent decisions for every fact:

| Decision | What LLM Figures Out |
|---|---|
| Concept mapping | Maps each hidden cell reference to the correct taxonomy concept |
| Context ID | Constructs the correct `contextRef` from entity + period + dimension combinations |
| Unit reference | Determines `unitRef` (INR, PURE, null) from concept data type |
| Pipe transformation | Decides if `\scale`, `\Date`, `\Boolean`, or `\findDecimalCount` is needed |
| Missing dimensions | Infers missing dimension mappings from surrounding context |

**Prompt Library (LLM-Agnostic — works with GPT-4, Claude, Gemini, Devin):**

| Prompt | Purpose |
|---|---|
| P001 | Taxonomy Analysis — validate taxonomy.json |
| P002 | Excel Spec Analysis — validate excel_spec.json |
| P003 | Template Generation — CORE prompt |
| P004 | Template Correction — self-correction loop |
| P005 | Root XML Generation — auto-generate namespace wrapper |

**Self-Correction Loop:**
```
Generate template
      ↓
Run XML conversion
      ↓
Compare vs expected output
      ↓
If match < 95% → feed errors to P004 → retry
      ↓
Repeat (max 3 iterations)
```

---

### Layer 4 — Validation & Output (Java — Existing Framework, UNCHANGED)

The existing `ExcelTextConvertor.java` (1,988 lines — untouched) takes:
- The `template.xml` (with `${Sheet<Cell>}` placeholders)
- The actual Excel file with real bank data

And replaces every placeholder with the real value, producing the final compliant XBRL XML ready for regulatory submission.

---

## Inputs Explained

### Taxonomy ZIP — Internal Structure

```
taxonomy.zip
│
├── META-INF/
│   └── taxonomyPackage.xml     ← Entry point — tells Python which report is which
│
├── core/
│   ├── rbi-core.xsd            ← ~18,540 concepts (field names + data types)
│   └── in-rbi-rep.xsd          ← ~9,456 report-specific concepts
│
└── reports/
    ├── R396/
    │   ├── entry.xsd           ← Entry point for this report
    │   ├── table-linkbase.xml  ← Table/row/column structure
    │   └── definition-linkbase.xml ← Dimensions and members
    ├── R322/
    └── R419/                   ← Hackathon demo report (112 facts, 18 contexts)
```

### Excel Spec — Visible vs Hidden

| Layer | Visibility | Contains |
|---|---|---|
| Main sheet (e.g., `BSR_Advances`) | Visible | Report layout — rows and columns a human sees |
| Column A | Hidden | Concept references for each row |
| Column B | Hidden | Dimension::member references |
| Column C | Hidden | `#TABLE#` markers + layout IDs |
| Hidden rows | Hidden | Column-axis concept references |
| `+Lineitems` sheet | veryHidden | All row-axis concepts |
| `+DynamicDomain` sheet | veryHidden | Dynamic dimension members |
| `StartUp` sheet | veryHidden | Date format, scale factor, decimal config |
| `Config` sheet | veryHidden | Report-level settings |
| `MainSheet` sheet | veryHidden | Master sheet-to-section mapping |

---

## Why LLM? Why Not Just Python/Java?

Custom code can handle simple, predictable cases — but real-world XBRL reports have edge cases that require *reasoning*, not just rules:

| Task | Python/Java | LLM |
|---|---|---|
| Read files, extract data | ✅ Perfect | ❌ Overkill |
| Simple 1:1 concept mapping | ✅ Easy | ❌ Overkill |
| Infer context naming convention | ❌ Too many variations | ✅ Understands patterns |
| Handle multi-dimensional combos | ❌ Exponential complexity | ✅ Reasons naturally |
| Pick correct pipe transformation | ❌ Needs hardcoded rules | ✅ Understands data types |
| Fill missing dimension mappings | ❌ Cannot infer | ✅ Infers from context |
| Adapt when taxonomy changes | ❌ Manual code update required | ✅ Reads new JSON, adapts automatically |
| Handle report-specific quirks | ❌ New code every time | ✅ Adapts automatically |

> **The LLM acts like a senior developer who has thoroughly read both JSONs** and knows how to connect all the dots intelligently — without you having to anticipate every possible edge case in code.

---

## Validated Results

| Report | Facts | Match Rate | Status |
|---|---|---|---|
| R396 | 284 facts | **100%** | ✅ Production validated |
| R322 | 856 facts | **89.1%** (763/856) | ✅ PoC validated |
| R419 | 112 facts | **Target ≥ 98%** | 🎯 Hackathon demo |

---

## Impact

| Metric | Before | After |
|---|---|---|
| Lines of code per report | 5,010 lines of Java | **Zero** |
| Time to onboard new report | 2–4 weeks | **Under 2 hours** |
| Reports scalable | 3 in 6 months | **100+ across 5 countries** |
| Developer dependency | 2–3 specialists | **Anyone with taxonomy + Excel** |
| Regression risk | High (blast radius) | **Zero — each report independent** |
| Per-report effort reduction | — | **87–95%** |

### Countries Supported

🇮🇳 India (RBI) · 🇸🇬 Singapore (MAS) · 🇭🇰 Hong Kong (HKMA) · 🇦🇺 Australia (APRA) · 🇹🇭 Thailand (BOT)

---

*Built at the Citi Hackathon — GenAI XGV Template Engine Team*