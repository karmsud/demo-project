# ABS Waterfall AI — System Architecture

**Date:** February 2026  
**Audience:** Distinguished Principal Engineer  
**Scope:** Complete architecture for an AI-powered ABS payment model generation platform

---

## 1. System Overview

ABS Waterfall AI is a 13-agent ecosystem that transforms raw legal deal documents (Pooling & Servicing Agreements, Indentures, Prospectus Supplements) into validated Python payment models. The system ingests, structures, compares, generates, validates, and monitors Asset-Backed Securities deals across a portfolio of 10,000+ deals spanning 500+ issuers and 36 years (1990–2026).

### 1.1 Core Value Proposition

Current state: Two humans independently build two Excel payment models, cross-check results, and pay investors. Future state: One human builds an Excel model, the AI builds a Python model, and they cross-check — cutting modeling effort by 50% while adding programmatic auditability, stress testing, and portfolio-wide intelligence.

### 1.2 Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **LLM** | VS Code GitHub Copilot Chat (Claude) | Zero API keys, zero incremental cost beyond Copilot subscription. All generation, analysis, and Q&A tasks route through Copilot Chat sub-agents |
| **Vector Store** | Chroma DB | Pip-installable, per-deal collection isolation, SQLite-backed, no external service |
| **Knowledge Graph** | NetworkX | Pip-installable, pickle/JSON serialization, in-memory traversal, no database server |
| **Embeddings** | `all-MiniLM-L6-v2` (sentence-transformers) | 384 dimensions, 80MB, free, sufficient for small per-deal corpora (~50-100 chunks) |
| **Runtime** | Python 3.13, local venv | No cloud services, no Docker, fully offline-capable |
| **Orchestration** | VS Code Tasks + CLI (argparse) | Three entry points: VS Code task runner, Copilot Chat, CLI |

### 1.3 Design Principles

1. **Deal-first mental model** — Users think in deals, not pipelines. Every operation starts with a deal ID.
2. **Deal isolation by construction** — Every file path, vector query, and graph traversal is scoped to one deal via the `DealScope` enforcer. Cross-deal access is read-only and limited to the Comparison Agent.
3. **Skills over monoliths** — Pure-function skills (parsers, embedders, comparators) are testable without LLM. Agents compose skills via a tool registry.
4. **Parallel, not phased** — All 13 agents share a common Foundation layer. Adding agent #13 is no harder than adding agent #2.
5. **No source, no answer** — Every generated value, formula, and constant must cite its governing document section. Hallucination is structurally prevented.

---

## 2. Pipeline Architecture

### 2.1 Two-Phase Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INGESTION PIPELINE (Stages 1-5)                │
│                                                                     │
│  Document   →  Section    →  Structured   →  Knowledge   →  Governing │
│  Preparation   Splitting     Extraction      Store Build    Doc Gen   │
│  (DOCX→MD)    (Regex+LLM)   (6 Parsers)    (Chroma+NX)   (Claude)   │
│                                                                     │
│  Output: 5 artifact groups per document                             │
│  Gate: All 5 must pass → "Ingestion Complete"                       │
└─────────────────────────────────────────────────────────────────────┘
                              │
                    Document Comparison Agent
                    (cross-portfolio matching,
                     teaching model selection)
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                     GENERATION PIPELINE (Stages 6-9)                │
│                                                                     │
│  Data Prep  →  Model Gen   →  Validation   →  Monthly Runs          │
│  (CSVs)       (Claude)       (vs Teaching)    (Production)          │
│                                                                     │
│  Output: payment_model.py + VALIDATION_NOTES.md + monthly outputs   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Stage Details

#### Stage 1: Document Preparation
- **Input:** PSA.docx (or .doc converted via PowerShell COM)
- **Process:** python-docx paragraph extraction with heading style detection; pdfplumber for PDF fallback; markdown table preservation
- **Output:** `full.md` + `full.txt` + `metadata.json` (includes content hash for dedup)

#### Stage 2: Section Splitting
- **Input:** `full.md`
- **Strategy:** Regex-first (ARTICLE/Section/EXHIBIT patterns) with configurable per-issuer section mapping tables. LLM fallback via Copilot Chat Claude for unclassifiable sections.
- **Output:** Named section files — `definitions.md`, `waterfall.md`, `accounts.md`, `loss_allocation.md`, `reporting.md`, `triggers.md`
- **Why regex-first:** Deals within the same shelf share consistent structure. Regex handles 90%+ of known issuers; LLM fallback catches 1990s-era formatting with zero API cost.

#### Stage 3: Structured Extraction
- **Input:** Section markdowns
- **Process:** Six production-tested parsers (Definitions, Waterfall, Accounts, LossAllocation, Reporting, CrossDocument) plus DealSetup and ClassesSetup extractors
- **Output:** JSON extractions (`definitions.json`, `waterfall_rules.json`, `accounts.json`, `loss_allocation.json`, `reporting_requirements.json`) + CSV data files (`deal_setup.csv`, `classes_setup.csv`)
- **Deterministic:** No LLM required. Pure regex + structural parsing.

#### Stage 4: Knowledge Store Build
- **Input:** Section markdowns + JSON extractions
- **Embedding:** Chunk section text at ~1000 chars at sentence boundaries → embed with `all-MiniLM-L6-v2` → upsert to per-deal Chroma collection
- **Graph:** Build NetworkX knowledge graph from JSON extractions with enriched schema (Deal, Section, Term, Rule, Account, Class, Trigger, Fee node types)
- **Output:** `chroma.sqlite3` (per-deal collection) + `deal_graph.pickle` + `deal_graph.json`
- **Multi-document merging:** When a deal has multiple documents (PSA + Indenture + ProSupp), vectors and graph nodes from all documents merge into a single deal-level store with source-document metadata.

#### Stage 5: Governing Document Generation
- **Input:** Sections + extractions
- **Process:** Claude generates enhanced markdown per section following a 4-layer format:
  1. **Legal Text** — verbatim from source
  2. **Interpretation** — plain English
  3. **Mathematical Formula** — where applicable
  4. **Code Hint** — Python pseudocode
- **Confidence tagging:** Interpretations below 80% confidence marked `⚠️ REVIEW NEEDED`
- **Scope:** Generated only from the payment source of truth document (user-designated).

#### Stage 6: Data Preparation
- CSV validation against schemas. Month 1 class balances auto-filled from `classes_setup.csv` original balances.

#### Stage 7: Payment Model Generation
- Claude generates `payment_model.py` following the Teaching Model structure with section-by-section generation, Sacred Rules enforcement, and full source citations.

#### Stage 8: Validation
- Run model on Month 1 data, compare per-class output to teaching output. Threshold: $0.01 max diff per class. Up to 3 self-healing iterations.

#### Stage 9: Monthly Production Runs
- Execute model with monthly inputs, produce output CSVs in `runs/month_N/`.

### 2.3 Ingestion Gate — Five Required Artifacts

A deal is NOT ready for model generation until all five artifact groups pass validation:

| # | Artifact | Validation |
|---|----------|-----------|
| 1 | Sectioned Markdown | ≥ 3 sections exist (definitions, waterfall, loss_allocation minimum) |
| 2 | Structured JSON Extractions | Schema-valid; definitions ≥ 50 terms; waterfall ≥ 10 rules |
| 3 | Deal Data CSVs | All required fields present; ≥ 2 certificate classes |
| 4 | Vector Store | Chroma collection exists with ≥ 20 vectors; test query returns results |
| 5 | Knowledge Graph | ≥ 50 nodes; connectivity check passes (no orphan nodes) |

---

## 3. Agent Ecosystem

### 3.1 Agent Design Philosophy

Each agent is a **tool-using LLM** with:
- A **system prompt** following the MISSION/CONTEXT/INPUTS/ACTIONS/OUTPUTS/VALIDATION structure (D11)
- A **tool allowlist** — only the skills it needs, registered via decorator-based `@agent_tool`
- A **DealScope injection** — every agent receives a deal scope that constrains all file/vector/graph access
- An **AgentBase** class providing the common infrastructure
- A **Quality Gate** — 5-dimension self-reflection scoring (Completeness, Accuracy, Citation Fidelity, Structural Conformance, Deal Scope Compliance) with all dimensions ≥ 8/10 required to pass (D6)
- A **Confidence-Based Autonomy** protocol — ≥90% auto-proceed, 66-89% flag for review, <66% halt and escalate (D7)
- A **Structured Escalation** handler — when blocked, agents produce machine-parseable escalation reports with type, context, solutions attempted, root blocker, impact, and recommended action (D9)
- A **State Persistence** layer — per-deal JSON state files for observations across runs (D12)

### 3.2 Agent Lifecycle Phases

```
Phase 1 — PRE-INGESTION
  └── Agent #1:  Document Quality Agent

Phase 2 — INGESTION
  └── Agent #2:  Document Ingestion Pipeline

Phase 3 — DEAL INTELLIGENCE
  ├── Agent #3:  Document Comparison Agent
  ├── Agent #4:  Deal Amendment Agent
  └── Agent #12: Deal Lifecycle Monitor Agent

Phase 4 — MODEL GENERATION
  ├── Agent #5:  Payment Model Creation Agent
  └── Agent #7:  Model Auditor Agent

Phase 5 — ANALYTICS & PROJECTIONS
  ├── Agent #8:  Stress Testing Agent
  ├── Agent #9:  Regression Testing Agent
  └── Agent #10: Cash Flow Projection Agent (+ Tax Sub-Agent)

Phase 6 — USER-FACING
  ├── Agent #11: Q&A / Research Agent
  └── Agent #13: Investor Reporting Agent

Day 2 (Parked)
  └── Agent #6:  Excel Payment Model Agent
```

### 3.3 Agent Detail: Document Quality Agent (#1)

**Role:** Pre-ingestion gatekeeper. Runs before any document enters the pipeline.

**Capabilities:**
- File readability validation (corrupt file detection)
- Text extractability check (minimum page count, language detection)
- **Document type auto-detection** — filename keyword analysis + first 10 pages content signature matching against 5 known types (PSA, Indenture, Prospectus Supplement, Trust Agreement, Servicing Agreement)
- **Duplicate prevention** — 3-layer detection:
  - Layer 1: SHA-256 content hash → exact duplicate → **REJECT**
  - Layer 2: MinHash + Jaccard on first 10 pages → near duplicate (>90%) → **WARN** with diff pages
  - Layer 3: TF-IDF cosine similarity on full text → content overlap (70-89%) → **INFO** (expected for related docs)
- Cross-portfolio duplicate scan (check against ALL deals, not just current)

**Output:** `doc_quality_report.json` + `doc_type_detection.json` per document

### 3.4 Agent Detail: Document Ingestion Pipeline (#2)

**Role:** Orchestrates Stages 1-5 sequentially. Produces the five artifact groups and validates via the ingestion gate.

**Tools:** Document converter, section splitter, all 6 parsers, embedder, graph builder, governing doc generator, ingestion validator.

**Output:** Fully populated deal folder with `ingestion_manifest.json` confirming readiness.

### 3.5 Agent Detail: Document Comparison Agent (#3)

**Role:** Cross-portfolio analyst. The **only agent** with read-only cross-deal access. Runs automatically after ingestion gate passes.

**Capabilities:**

| Capability | Description |
|------------|-------------|
| **Closest Deal Matching** | Compare definition sets, waterfall structures, class configs, trigger logic across all ingested deals. Produce ranked similarity scores. |
| **Delta Report** | Section-by-section diff vs closest match: identical / modified / new / removed |
| **Teaching Model Selection** | Recommend best existing `payment_model.py` as base for the new deal. Within a series, the previous deal's model becomes the next deal's template. |
| **New Concept Detection** | Flag definitions or rules that don't exist in ANY deal in the portfolio — genuinely novel structures requiring human review |
| **Language Quality Analysis** | Compare ambiguous/contradictory language against clearer versions of the same concept in other deals — risk mitigation at deal inception |
| **Portfolio Intelligence** | Answer queries like "Which deals have this trigger structure?" or "How many deals use cross-group allocation?" |

**Similarity Scoring:**
```
Overall Similarity = weighted average of:
  0.30 × definition_overlap      (Jaccard on term names + semantic on meanings)
  0.25 × waterfall_structure      (rule count, hierarchy depth, payment targets)
  0.20 × class_configuration      (number of classes, group structure, types)
  0.15 × trigger_logic            (trigger types, thresholds, stepdown conditions)
  0.10 × account_structure        (account names, flow rules)
```

**Output Artifacts:** `comparison/closest_match.json`, `delta_report.json`, `language_flags.json`, `teaching_model_recommendation.json`

**Deal Scoping Exception Rules:**
- ✅ CAN read any deal's `extractions/*.json` and `models/payment_model.py`
- ✅ CAN search across all deal vector stores
- ❌ CANNOT write to any deal folder other than the active deal
- ❌ CANNOT modify another deal's artifacts

### 3.6 Agent Detail: Deal Amendment Agent (#4)

**Role:** Process amendments to existing deals. Amendments create versioned layers — originals are never overwritten.

**Capabilities:**
- Amendment ingestion with changed-section extraction
- Version diffing (original vs amended sections)
- Cascade detection — identify which downstream artifacts need regeneration (extractions, governing docs, model)
- Snapshot original before amendment (write to `amendments/v1_original/`)
- Re-extraction trigger on amended sections only

**Version Chain:**
```
amendments/
  v1_original/         ← Snapshot of pre-amendment extractions
  v2_amendment_2024_01/ ← Amended extractions
  active/              ← Points to current active version
```

### 3.7 Agent Detail: Payment Model Creation Agent (#5)

**Role:** Generate a working `payment_model.py` from governing docs, CSV data, and (optionally) a comparison-recommended teaching model.

**Tools:** `read_governing_doc`, `read_csv`, `vector_search`, `graph_query`, `validate_model`

**Generation Strategy:**
1. Read all governing docs + CSVs
2. Plan model structure (constants → rate caps → interest → triggers → principal → OC → CE)
3. Generate code section-by-section (not monolithically)
4. Self-validate against teaching output
5. If validation fails, read diff report, fix, re-validate (up to 3 iterations)

**Output Structure:**
```python
# payment_model.py — Generated for {deal_name}
# --- CONSTANTS --- (with Source: citations)
# --- DATA LOADING ---
# --- RATE CALCULATIONS ---
# --- INTEREST WATERFALL ---
# --- TRIGGERS ---
# --- PRINCIPAL DISTRIBUTION ---
# --- OC / EXCESS CASHFLOW ---
# --- MAIN ---
```

**Sacred Rules (enforced in system prompt):**
- Every constant must cite its governing document section
- Every formula must match the governing doc's mathematical representation
- No hardcoded values without source
- Output structure must match the teaching model template
- CSV column names must match defined schemas

### 3.8 Agent Detail: Excel Payment Model Agent (#6) — Day 2

**Status:** Parked for Day 2. Interface designed for compatibility.

**Approach:** Template-based openpyxl generation. Replicate `payment_model.py` logic in Excel formulas with cell-level mapping. Cross-validate: Excel output must match Python output within $0.01.

### 3.9 Agent Detail: Model Auditor Agent (#7)

**Role:** Independent post-generation audit. Verifies every constant, formula, and business rule in `payment_model.py` traces back to a governing document.

**Audit Checks:**
- Every constant sourced (no magic numbers)
- Every formula matches governing doc math
- No hardcoded values without citation
- Waterfall priority order matches governing doc sequence
- Class names match `classes_setup.csv`
- Trigger thresholds match definitions

**Output:** `audit_report.json` — pass/fail per check with governing doc citations

### 3.10 Agent Detail: Stress Testing Agent (#8)

**Role:** Design and execute governing-doc-aware stress scenarios.

**Key Innovation:** Unlike generic stress testing, this agent reads the deal's trigger thresholds from governing docs and designs boundary scenarios that specifically test trigger transitions.

**Capabilities:**
- Pre-built scenario library (high CDR, rapid prepay, rate spike, etc.)
- Governing doc integration — read trigger thresholds to design boundary scenarios
- Scenario execution via cashflow engine
- Stress report: which scenarios trip triggers, which classes take losses

### 3.11 Agent Detail: Regression Testing Agent (#9)

**Role:** Capture "known good" model outputs as baselines and detect drift after any change.

**Triggers:** Runs automatically after model regeneration, amendment processing, or config changes.

**Output:** `regression_results.json` — pass/fail per month per class, with diff details for failures.

### 3.12 Agent Detail: Cash Flow Projection Agent (#10) + Tax Sub-Agent

**Role:** Multi-month projections with scenario-based assumptions.

**Cashflow Engine (shared skill):** Runs `payment_model.py` across N months with injected scenario parameters (CPR, CDR, severity, delinquency overrides). Collects results into time-series DataFrames.

**Tax Sub-Agent:** Computes OID calculations and generates data for 8-K/10-K tax filings. The tax team uses projections for these filings — this agent produces the required output format.

**Output:** `projections/base_case.csv`, `projections/stress.csv`

### 3.13 Agent Detail: Q&A / Research Agent (#11)

**Role:** Answer questions about a specific deal using RAG retrieval scoped to that deal's knowledge store.

**Tools:** `vector_search`, `graph_query`, `read_extraction`, `read_governing_doc`, `read_csv`

**Policy:** Every answer cites source (file path + section). If information not found, explicitly states so — never hallucinate. Multi-document awareness: searches across all documents in a deal.

### 3.14 Agent Detail: Deal Lifecycle Monitor Agent (#12)

**Role:** Ongoing deal surveillance and alert generation.

**Capabilities:**
- Trigger status tracking month-over-month
- Clean-up/call date monitoring — flag approaching maturity events
- Stepdown detection (pre-stepdown → stepdown transitions)
- Alert generation with severity levels and recommended actions

**Output:** `lifecycle_alerts.json`

### 3.15 Agent Detail: Investor Reporting Agent (#13)

**Role:** Generate investor reports from payment model outputs ONLY.

**Scope Boundary:**
- **IN SCOPE:** Class-level payments, trigger status, OC levels, CE calculations, distribution amounts — anything the payment model produces
- **OUT OF SCOPE:** Collateral-level reporting (handled by existing 20-30 year mature collateral processing software)

**Process:** Map `reporting_requirements.json` entries to available model outputs → assemble data → invoke report generator (Excel/PDF/Markdown) → coverage check (identify requirements that cannot be fulfilled from model data alone).

---

## 4. Data Architecture

### 4.1 Deal Folder Structure

```
deals/
  bear_stearns_2006_he2/
    deal_manifest.json              ← Deal metadata + payment_source_of_truth
    documents/                      ← ALL source documents
      psa/                          ← Payment source of truth
        source.docx
        full.md
        full.txt
        metadata.json               ← Includes content_hash for dedup
        doc_type_detection.json     ← Auto-detected document type
        sections/
          definitions.md
          waterfall.md
          accounts.md
          loss_allocation.md
          reporting.md
          triggers.md
        extractions/
          definitions.json
          waterfall_rules.json
          accounts.json
          loss_allocation.json
          reporting_requirements.json
      indenture/                    ← Additional document (same sub-structure)
        source.docx
        full.md
        metadata.json
        doc_type_detection.json
        sections/
        extractions/
      prospectus_supplement/        ← Another document (same sub-structure)
    data/
      deal_setup.csv
      classes_setup.csv
      month_1/
        monthly_input.csv
        class_balances.csv
        prior_shortfalls.csv
        output_teaching.csv         ← For validation
    vectorstore/                    ← Combined vectors from ALL documents
      chroma.sqlite3
    graph/                          ← Combined graph from ALL documents
      deal_graph.pickle
      deal_graph.json
    governing_docs/                 ← Generated from payment source of truth ONLY
      01_definitions.md
      02_waterfall.md
      03_loss_allocation.md
      04_triggers.md
    models/
      payment_model.py
      VALIDATION_NOTES.md
    runs/
      month_1/
        output.csv
        run_metadata.json
    comparison/                     ← Document Comparison Agent outputs
      closest_match.json
      delta_report.json
      language_flags.json
      teaching_model_recommendation.json
    amendments/                     ← Deal Amendment Agent outputs
      v1_original/
      v2_amendment_2024_01/
      active/
    ingestion_manifest.json         ← Ingestion status + artifact inventory
```

### 4.2 Deal Manifest Schema

Each deal has a `deal_manifest.json` at its root:

```json
{
  "deal_id": "bear_stearns_2006_he2",
  "deal_name": "Bear Stearns Asset Backed Securities I Trust 2006-HE2",
  "issuer": "Bear Stearns",
  "series": "2006-HE2",
  "shelf": "ABS Series",
  "closing_date": "2006-03-01",
  "payment_source_of_truth": "psa",
  "documents": {
    "psa": {
      "original_filename": "...",
      "detected_type": "psa",
      "detection_confidence": 0.95,
      "content_hash": "a1b2c3d4...",
      "ingestion_status": "complete",
      "is_payment_source": true
    },
    "indenture": {
      "original_filename": "...",
      "detected_type": "indenture",
      "detection_confidence": 0.92,
      "content_hash": "e5f6g7h8...",
      "ingestion_status": "complete",
      "is_payment_source": false
    }
  },
  "amendment_history": [],
  "portfolio_comparison": {
    "closest_match_deal": null,
    "similarity_score": null,
    "comparison_run_at": null
  }
}
```

### 4.3 JSON Extraction Schemas

#### definitions.json
```json
[{
  "term": "Closing Date",
  "definition": "March 1, 2006",
  "source_section": "Article I",
  "value": "2006-03-01",
  "formula": null,
  "dependencies": [],
  "category": "date"
}]
```

Categories: `constant`, `rate`, `date`, `formula`, `entity`, `account`, `threshold`, `other`

#### waterfall_rules.json
```json
[{
  "rule_id": "interest_1",
  "priority": 1,
  "hierarchy": "(a)(1)(i)",
  "description": "Pay interest to Class I-A-1...",
  "rule_type": "interest",
  "source_section": "Section 5.04",
  "pays_to": ["I-A-1"],
  "conditions": [],
  "depends_on": ["Net Rate Cap"],
  "sub_rules": []
}]
```

Rule types: `interest`, `principal`, `loss_allocation`, `fee`, `reserve`, `overcollateralization`, `other`

#### accounts.json
```json
[{
  "account_name": "Distribution Account",
  "purpose": "Collects all available funds for distribution",
  "source_section": "Section 5.06",
  "initial_deposit": null,
  "flow_rules": ["interest_1", "principal_1"]
}]
```

### 4.4 Knowledge Graph Schema

```
Node Types:
  Deal        — One per deal
  Section     — definitions, waterfall, accounts, etc.
  Term        — Defined terms with value/formula/category
  Rule        — Waterfall rules with priority/type
  Account     — Named accounts with purpose/flow rules
  Class       — Certificate classes with balance/margin/group
  Trigger     — Trigger events with condition/effect
  Fee         — Fees (servicing, trustee, etc.)

Edge Types:
  CONTAINS    — Deal → Section, Section → Term
  DEFINES     — Section → Term
  DEPENDS_ON  — Term → Term (cross-references)
  REFERENCES  — Rule → Term
  PAYS_TO     — Rule → Class or Account
  FLOWS_INTO  — Account → Account
  TRIGGERS    — Trigger → Rule (enables/disables)
  CALC_BY     — Term → Formula/Rule
  BELONGS_TO  — Class → Group (loan group)
  ALLOC_TO    — LossRule → Class
```

### 4.5 Embedding Strategy

**What gets embedded:**
- Each section markdown, chunked at ~1000 chars at sentence boundaries
- Each JSON definition entry as a standalone chunk (term name + definition text)
- Each waterfall rule as a standalone chunk (rule text + priority + dependencies)
- NOT raw CSVs (structured data, not for semantic search)

**Metadata per vector:** `{deal_id, section_type, source_file, source_document, chunk_index}`

**Isolation:** One Chroma collection per deal (`collection_name = deal_id`). At 10,000 deals, a shared collection with 1M+ vectors is unworkable. Per-collection gives O(1) deal isolation, easy deletion, and simple backup.

### 4.6 Document Type Detection Signatures

```python
DOC_TYPE_SIGNATURES = {
    "psa": {
        "filename_keywords": ["pooling", "servicing agreement", "psa"],
        "content_signatures": [
            "pooling and servicing agreement", "distribution date",
            "certificateholders", "available funds", "waterfall"
        ]
    },
    "indenture": {
        "filename_keywords": ["indenture", "trust indenture"],
        "content_signatures": [
            "indenture", "trustee", "noteholders",
            "events of default", "collateral"
        ]
    },
    "prospectus_supplement": {
        "filename_keywords": ["prospectus", "prosup", "prosupp", "supplement"],
        "content_signatures": [
            "prospectus supplement", "risk factors",
            "description of the certificates", "the mortgage pool"
        ]
    },
    "trust_agreement": {
        "filename_keywords": ["trust agreement", "declaration of trust"],
        "content_signatures": [
            "declaration of trust", "trust estate", "beneficial interests"
        ]
    },
    "servicing_agreement": {
        "filename_keywords": ["servicing agreement", "sub-servicing"],
        "content_signatures": [
            "servicing agreement", "servicer",
            "servicing duties", "collection procedures"
        ]
    }
}
```

---

## 5. Deal Scoping Contract

The single most important safety mechanism. Every operation — file read, vector query, graph traversal — is confined to one deal.

### 5.1 Six Rules

| Rule | Enforcement |
|------|------------|
| **PATH ALLOWLIST** | Every file operation resolves to `deals/<current_deal_id>/`. Paths escaping this boundary are rejected. |
| **VECTOR ISOLATION** | Every vector query uses a per-deal Chroma collection (`collection_name = deal_id`). |
| **GRAPH ISOLATION** | Load ONLY the graph from `deals/<current_deal_id>/graph/deal_graph.pickle`. Never load multiple deal graphs simultaneously. |
| **LLM CONTEXT BOUNDARY** | System prompt includes: "You are working on deal: {deal_id}. Do NOT reference any other deal." |
| **OUTPUT CONTAINMENT** | All generated files go into `deals/<current_deal_id>/`. Never write outside the deal folder. |
| **MISMATCH REJECTION** | If a tool call references a deal_id different from the active deal, raise `DealScopingViolation`. |

### 5.2 DealScope Enforcer

```python
class DealScope:
    def __init__(self, deal_id: str, deals_root: Path):
        self.deal_id = deal_id
        self.deal_path = deals_root / deal_id

    def resolve(self, relative_path: str) -> Path:
        """Resolve a path within the deal folder. Raises on escape."""
        resolved = (self.deal_path / relative_path).resolve()
        if not str(resolved).startswith(str(self.deal_path.resolve())):
            raise DealScopingViolation(f"Path escapes deal boundary: {relative_path}")
        return resolved

    def get_vector_collection(self) -> str:
        return self.deal_id

    def get_graph_path(self) -> Path:
        return self.deal_path / "graph" / "deal_graph.pickle"
```

### 5.3 Cross-Deal Read-Only Mode

The Document Comparison Agent uses `DealScope.read_only(other_deal_id)` — a restricted scope that permits reading extractions and models from other deals but blocks all write operations.

---

## 6. Agent Interaction Architecture

### 6.1 Request Flow

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐
│  User     │────▶│ Deal Scoping  │────▶│ Agent Router │
│  Request  │     │ Contract      │     │              │
└──────────┘     └───────────────┘     └──────┬───────┘
                                              │
          ┌────────────┬────────────┬─────────┼─────────┬────────────┐
          ▼            ▼            ▼         ▼         ▼            ▼
   ┌────────────┐ ┌─────────┐ ┌─────────┐ ┌──────┐ ┌───────┐ ┌─────────┐
   │ Doc Quality│ │Ingestion│ │Comparison│ │Model │ │  Q&A  │ │Analytics│
   │ Agent (#1) │ │Pipeline │ │Agent (#3)│ │Gen   │ │Agent  │ │Agents   │
   │            │ │(#2)     │ │READ-ONLY │ │(#5)  │ │(#11)  │ │(#8-10)  │
   └─────┬──────┘ └────┬────┘ └────┬─────┘ └──┬───┘ └───┬───┘ └────┬────┘
         │              │          │           │         │          │
         └──────────────┴──────────┴───────────┴─────────┴──────────┘
                                   │
                    ┌──────────────────────────────┐
                    │         SKILL LAYER           │
                    │  (Pure functions, deal-scoped) │
                    └──────────────┬───────────────┘
                                  │
                   ┌──────────────┼──────────────┐
                   ▼                             ▼
            ┌─────────────┐            ┌──────────────────┐
            │ DEAL FOLDER │            │ PORTFOLIO         │
            │ (read/write)│            │ (read-only, #3)   │
            └─────────────┘            └──────────────────┘
```

### 6.2 Automatic Pipeline Flow

```
Document dropped
      │
      ▼
Document Quality Agent (#1)  ─── REJECT if corrupt/duplicate
      │ PASS
      ▼
Ingestion Pipeline (#2)      ─── Stages 1-5
      │ Ingestion Gate ✅
      ▼
Document Comparison Agent (#3) ─── Find closest deal, select teaching model
      │
      ▼
Model Creation Agent (#5)    ─── Generate payment_model.py
      │
      ▼
Model Auditor Agent (#7)     ─── Verify all formulas cite sources
      │
      ▼
Validation (Stage 8)         ─── Compare output vs teaching within $0.01
      │ PASS
      ▼
Monthly Runs (Stage 9)       ─── Production execution
```

### 6.3 On-Demand Agents

| Agent | Trigger |
|-------|---------|
| Q&A Agent (#11) | User asks a question in Copilot Chat |
| Stress Testing (#8) | User requests stress scenarios |
| Cash Flow Projection (#10) | User requests multi-month projection |
| Investor Reporting (#13) | User requests investor report |
| Deal Lifecycle Monitor (#12) | Scheduled or on-demand surveillance |
| Regression Testing (#9) | Automatic after model regeneration/amendment |
| Deal Amendment (#4) | User ingests an amendment document |

---

## 7. Skill Layer

Skills are pure Python functions testable without any LLM. They form the computational backbone that agents compose via the tool registry.

### 7.1 Skill Inventory

| Skill | Source | Purpose |
|-------|--------|---------|
| `parsers.py` | Reused (v2) | 6 parsers: Definitions, Waterfall, Accounts, LossAllocation, Reporting, CrossDocument |
| `deal_setup_extractor.py` | Reused (v2) | DealSetupExtractor + ClassesSetupExtractor |
| `embedder.py` | New (refactored from v1) | Chunk text, embed with all-MiniLM-L6-v2, upsert to Chroma |
| `graph_builder.py` | Extended (from v2) | Build enriched NetworkX graph with Class/Trigger/Fee nodes |
| `vector_search.py` | New (refactored from v1) | Deal-scoped Chroma query with metadata filters |
| `deal_comparator.py` | New | Cross-deal comparison: definitions, waterfall, classes, aggregate similarity |
| `document_hasher.py` | New | SHA-256 content hash + MinHash near-duplicate detection |
| `document_classifier.py` | New | Document type detection from filename + content signatures |
| `csv_validator.py` | New | CSV schema validation |
| `output_comparator.py` | New | Model output comparison, diff reports |
| `amendment_manager.py` | New | Version tracking, amendment chain management |
| `cashflow_engine.py` | New | Multi-month model execution with scenario parameter injection |
| `report_generator.py` | New | Report formatting: Excel (openpyxl), PDF, Markdown |

### 7.2 Tool Registry

```python
@agent_tool(name="vector_search", agents=["qa", "model_creation", "comparison"])
def vector_search(query: str, deal_scope: DealScope) -> list[dict]:
    """Search deal-scoped Chroma collection."""
    ...
```

Each agent declares a tool allowlist. The framework enforces that agents cannot access tools outside their allowlist.

---

## 8. Orchestration Layer

### 8.1 Three Entry Points

| Entry Point | Mechanism | Example |
|-------------|-----------|---------|
| **VS Code Task** | `tasks.json` → `python main.py <command>` | Ctrl+Shift+P → "ABS: Ingest Deal" |
| **Copilot Chat** | Natural language → agent routing | `@abs What's the OC target for HE2?` |
| **CLI** | `python main.py <command> --deal <id>` | `python main.py ingest --deal bear_stearns_2006_he2` |

### 8.2 CLI Commands

| Command | Purpose |
|---------|---------|
| `ingest --deal <id>` | Run full ingestion pipeline (Stages 1-5) |
| `generate --deal <id>` | Generate payment model (Stages 6-8) |
| `validate --deal <id> --month <n>` | Validate model against teaching |
| `run --deal <id> --month <n>` | Run monthly distribution (Stage 9) |
| `status --deal <id>` | Check ingestion status |
| `compare --deal <id>` | Run comparison agent |
| `amend --deal <id>` | Process amendment |

### 8.3 Copilot Instructions

A `copilot-instructions.md` at workspace root governs Copilot Chat behavior:
- Every operation is deal-scoped
- Sources must be cited
- Follows Sacred Rules for code generation
- References governing docs, not raw sections

---

## 9. Testing Strategy

### 9.1 Four Testing Layers

| Layer | What | How |
|-------|------|-----|
| **Unit** | Skills (parsers, embedders, comparators) | Known input → expected output. HE2 as primary fixture. |
| **Integration** | Pipeline stages (converter → splitter → extractor → knowledge store → governing docs) | Stage N output feeds Stage N+1. Assert artifact quality. |
| **Agent** | Agent behavior (each of 13 agents) | Agent makes correct tool calls, produces correct outputs, respects deal scope. |
| **End-to-End** | Full pipeline + cross-deal isolation | HE2 DOCX → validated model within $0.01. Two deals ingested, no leakage. |

### 9.2 Primary Test Fixture

**Bear Stearns 2006-HE2** — chosen because we have:
- Full PSA.docx
- Pre-extracted .txt files (for comparison)
- Complete Teaching Model with validated output
- `VALIDATION_NOTES.md` proving 0.0000 diff across all 20 certificate classes

### 9.3 Exit Criteria

| # | Criterion |
|---|-----------|
| 1 | HE2 ingests end-to-end from DOCX |
| 2 | Duplicate document rejected on re-ingestion |
| 3 | Document type correctly detected for PSA |
| 4 | Payment model generates within $0.01 of teaching |
| 5 | Model auditor validates all formulas cite sources |
| 6 | Q&A answers 10 test questions with citations |
| 7 | Cross-deal isolation holds (HE1 + HE2) |
| 8 | Comparison agent finds HE1 as closest to HE2 |
| 9 | Amendment creates version chain without corruption |
| 10 | Cash flow projection runs 12 months |
| 11 | Stress test trips triggers at expected thresholds |
| 12 | Regression detects model drift |
| 13 | Investor report generated in Excel |
| 14 | Lifecycle monitor alerts on trigger breach |
| 15 | CLI `--help` works for all subcommands |

---

## 9A. Quality Assurance Architecture

### 9A.1 Quality Gate Infrastructure

Every agent output passes through a standardized quality gate before acceptance. This is enforced at the `AgentBase` level — no agent can bypass it.

**Five Scoring Dimensions:**

| Dimension | Description | Threshold |
|-----------|-------------|-----------|
| Completeness | All expected fields/sections present | ≥ 8/10 |
| Accuracy | Values match source documents | ≥ 8/10 |
| Citation Fidelity | Every value traces to a governing doc section | ≥ 8/10 |
| Structural Conformance | Output matches declared schema/contract | ≥ 8/10 |
| Deal Scope Compliance | No cross-deal data contamination | 10/10 (binary) |

**Retry Protocol:** On failure, the agent receives its scores + dimension-specific feedback. Maximum 3 retries before structured escalation.

### 9A.2 Confidence-Based Autonomy

All LLM-generated outputs carry a confidence score:

| Confidence | Action | Processing |
|------------|--------|------------|
| ≥ 90% | Auto-proceed | No human intervention |
| 66-89% | Flag for review | Output accepted but marked `⚠️ REVIEW NEEDED` |
| < 66% | Halt + escalate | Processing stops, escalation report generated |

### 9A.3 Output Contracts with Standardized IDs

Every extraction artifact uses traceable ID prefixes:

- `DEF-001` through `DEF-NNN` for definitions
- `RULE-001` through `RULE-NNN` for waterfall rules
- `ACC-001` through `ACC-NNN` for accounts
- `LOSS-001` through `LOSS-NNN` for loss allocation rules
- `TRIG-001` through `TRIG-NNN` for triggers
- `SEC-001` through `SEC-NNN` for sections
- `RPT-001` through `RPT-NNN` for reporting requirements

These IDs enable traceability from generated `payment_model.py` constants back to their source governing document sections.

### 9A.4 Structured Escalation Protocol

When an agent cannot resolve an issue after quality gate retries:

```json
{
    "escalation_type": "<category>",
    "agent": "<agent_name>",
    "deal_id": "<deal>",
    "context": "<what was being attempted>",
    "solutions_attempted": ["<list of approaches tried>"],
    "root_blocker": "<fundamental issue>",
    "impact": "<what cannot proceed>",
    "recommended_action": "<suggested resolution>"
}
```

### 9A.5 Generate-Evaluate-Refine Loop

Critical extraction and generation stages use iterative refinement:

1. **Generate** — Initial extraction/generation pass
2. **Evaluate** — Score output against quality gate dimensions
3. **Refine** — Agent receives failing scores + specific feedback, produces improved output
4. **Converge** — Stop when: all scores ≥ 8/10 OR two consecutive iterations produce identical output OR max retries reached

Applied to: Stage 3 (Structured Extraction), Stage 5 (Governing Doc Gen), Stage 7 (Model Gen).

### 9A.6 Adversarial Pre-Analysis

Before generating a payment model (Stage 7), run a pre-mortem analysis:
- Identify ambiguous definitions
- Flag unusual waterfall conditions
- Detect potential calculation edge cases (e.g., 360/365 day count, negative balances)

Output feeds into the Model Creation Agent's context as risk-awareness priming.

---

## 10. Reference Architecture

### 10.1 Teaching Model Evolution

Teaching models are not static. Within a deal series:

```
WHFSC 2024-1 → manual teaching model (first in series)
WHFSC 2024-2 → uses 2024-1 as base (comparison: 0.96 match)
WHFSC 2024-3 → uses 2024-2 as base (comparison: 0.95 match)
WHFSC 2025-1 → uses 2024-3 as base
```

Teaching models accumulate in:
```
reference/
  Teaching_Model_Template/       ← Original gold standard (HE2, manually validated)
  teaching_models/
    whfsc_2024_1/payment_model.py
    whfsc_2024_2/payment_model.py  ← Generated using 2024-1 as base
    ...
```

### 10.2 Multi-Document Deal Design

Deals can have 10+ documents. Key decisions:
- **User specifies** which document is the payment source of truth (stored as `payment_source_of_truth` in `deal_manifest.json`)
- **ALL documents** are ingested with full 5-artifact treatment
- **ALL documents** feed the unified vectorstore and knowledge graph (with source-document metadata)
- **ONLY the payment source of truth** feeds governing doc generation and model creation
- **All documents** are available for Q&A, comparison, and analysis

### 10.3 Portfolio Scale Considerations

| Parameter | Scale |
|-----------|-------|
| Deals | 10,000+ |
| Issuers | 500+ |
| Date range | 1990–2026 |
| Documents per deal | 10+ |
| Total artifacts at full scale | ~500,000 |

**Implications:**
- Per-deal Chroma collections — shared collection at this scale is impossible
- Section mapping tables must be learned/accumulated per issuer, not just manually configured
- Document Comparison Agent becomes exponentially more valuable as portfolio grows
- Format-adaptive section splitter with per-issuer overrides is essential for 36 years of formatting variation

---

## 11. Pipeline Code Structure

```
pipeline/
  __init__.py
  cli.py                            ← CLI entry point
  deal_scope.py                     ← DealScope enforcer
  deal_manifest.py                  ← DealManifest loader/validator
  ingestion/
    document_converter.py            ← Stage 1
    document_intelligence.py         ← Type detection + duplicate prevention
    section_splitter.py              ← Stage 2
    structured_extractor.py          ← Stage 3
    knowledge_store.py               ← Stage 4
    governing_doc_generator.py       ← Stage 5
    ingestion_validator.py           ← Gate check
  generation/
    data_prep.py                     ← Stage 6
    model_generator.py               ← Stage 7
    model_validator.py               ← Stage 8
    model_runner.py                  ← Stage 9
  agents/
    agent_base.py                    ← Base class
    agent_tools.py                   ← Tool registry
    document_quality_agent.py        ← #1
    comparison_agent.py              ← #3
    deal_amendment_agent.py          ← #4
    model_creation_agent.py          ← #5
    model_auditor_agent.py           ← #7
    stress_testing_agent.py          ← #8
    regression_testing_agent.py      ← #9
    cashflow_projection_agent.py     ← #10
    qa_agent.py                      ← #11
    deal_lifecycle_agent.py          ← #12
    investor_reporting_agent.py      ← #13
  skills/
    parsers.py                       ← From v2 (as-is)
    deal_setup_extractor.py          ← From v2 (as-is)
    embedder.py
    graph_builder.py
    vector_search.py
    deal_comparator.py
    document_hasher.py
    document_classifier.py
    csv_validator.py
    output_comparator.py
    amendment_manager.py
    cashflow_engine.py
    report_generator.py
  config/
    section_maps.py
    schemas.py
    doc_type_signatures.py
    constants.py
```

---

## Appendix A: Build vs Reuse Summary

| Component | Action |
|-----------|--------|
| 6 Parsers | Reuse as-is (v2) |
| DealSetupExtractor + ClassesSetupExtractor | Reuse as-is (v2) |
| Document Converter | Minor refactor (v2) |
| GraphBuilder | Extend with Class/Trigger/Fee nodes (v2) |
| Embedder | Refactor — swap OpenAI → all-MiniLM-L6-v2 (v1) |
| Vector Search | Refactor — add deal scoping (v1) |
| Teaching Model Template | Keep in `reference/` |
| Section Splitter | Build new |
| Governing Doc Generator | Build new |
| DealScope Enforcer | Build new |
| All 13 Agents | Build new |
| Document Intelligence | Build new |
| CLI + VS Code Tasks | Build new |
| Cross-portfolio skills | Build new |
| Amendment Manager | Build new |
| Cashflow Engine | Build new |
| Report Generator | Build new |

**Estimated scope:** ~8,900 new lines + ~2,800 reused = ~11,700 total across ~88 files
