# ABS Waterfall AI — Executive Overview

**Date:** February 2026  
**Audience:** Senior Business Line Leader  
**Prepared by:** ABS Technology Team

---

## The Vision

Today, building a payment model for a single ABS deal requires two analysts building two independent Excel models, cross-checking results, and manually tracing every formula back to a 300+ page legal document. This process takes days per deal across a portfolio of 10,000+ deals.

**ABS Waterfall AI** replaces one of those two analysts with an AI system that reads the legal documents, understands the payment waterfall, and generates a validated Python payment model — automatically, in minutes, with every formula traced to its source paragraph.

The human analyst still builds their Excel model. The AI builds the Python model. They cross-check each other. **Same rigor, half the effort, and a new layer of programmatic auditability that Excel cannot provide.**

---

## What Problem Are We Solving?

| Pain Point Today | How ABS Waterfall AI Solves It |
|-----------------|-------------------------------|
| **Days to build one model** — Analysts manually read 300+ page PSAs, find every definition, trace every formula, code it in Excel | AI reads the full document in seconds, extracts all definitions, rules, and formulas, and generates a working model with source citations |
| **No institutional memory** — When a new deal in the same series arrives, the analyst starts from scratch or informally remembers what the last deal looked like | The system automatically compares every new deal against the full portfolio of 10,000+ deals, finds the closest match, and tells you exactly what changed |
| **Risk from ambiguous language** — Legal drafting quality varies wildly across 500+ issuers and 36 years. Ambiguities in payment waterfalls cause disputes. | AI flags poorly drafted or contradictory language by comparing against clearer versions of the same concept in other deals — catching problems at inception |
| **No stress testing** — Running "what if" scenarios requires manual recalculation in Excel for every scenario | AI runs unlimited stress scenarios against governing document trigger thresholds, automatically identifying which scenarios trip triggers and which classes take losses |
| **No regression safety** — When a deal is amended or a model is updated, there's no systematic check that nothing broke | AI captures baselines and automatically detects any drift in outputs after changes |
| **Audit trail** — "Why does this formula say 3.65%?" requires hunting through the PSA | Every constant, formula, and business rule in the AI model cites the exact governing document section it came from |

---

## The 13-Agent Ecosystem

ABS Waterfall AI is not one tool — it's an ecosystem of 13 specialized AI agents, each purpose-built for a specific part of the deal lifecycle. Here's what each one does, why it matters, and why we build it this way.

---

### Agent #1: Document Quality Agent
**What it does:** Before any document enters the system, this agent checks that it's readable, not corrupted, and not a duplicate of something already ingested.

**Why it matters:** Garbage in, garbage out. If a duplicate PSA gets ingested, every downstream agent — Q&A, comparison, model generation — produces confused or duplicated results. This agent prevents that at the front door.

**How it works:** Checks file integrity, auto-detects document type (PSA vs Indenture vs Prospectus Supplement), and runs a 3-layer duplicate check (exact match, near-duplicate, content overlap) against every document already in the portfolio.

---

### Agent #2: Document Ingestion Pipeline
**What it does:** Takes a raw legal document (DOCX/PDF) and transforms it into structured, queryable knowledge — split into named sections, parsed into definitions and rules, embedded into a searchable vector store, and organized into a knowledge graph.

**Why it matters:** This is the foundation everything else depends on. Without structured extraction, no agent can answer questions, no model can be generated, no comparison can be run. It turns a 300-page legal document into actionable data.

**How it works:** Five automated stages: Document Conversion → Section Splitting → Structured Extraction → Knowledge Store Build → Governing Document Generation. The last stage produces enhanced "governing docs" — the bridge between raw legal text and code — where every legal clause is paired with its plain English interpretation, mathematical formula, and code hint.

---

### Agent #3: Document Comparison Agent
**What it does:** After ingestion, this agent compares the new deal against the entire portfolio of previously ingested deals. It finds the closest matching deal, produces a section-by-section diff, flags new concepts with no precedent, and recommends which existing model to use as a starting point for the new deal.

**Why it matters:** This is the "institutional memory" agent. In a portfolio of 10,000+ deals across 500+ issuers:
- **For modeling:** It identifies a 90%+ matching deal from the same series and says "start from this model, change these 3 constants" — turning days of work into minutes.
- **For relationship managers:** It shows exactly what's different in a new deal versus its predecessor, instantly.
- **For risk:** It flags genuinely novel structures (definitions or rules that don't exist anywhere in the portfolio) that need human review.
- **For legal quality:** It compares ambiguous language against clearer versions in other deals, catching drafting problems before they surface in payment disputes.

**How it works:** Weighted similarity scoring across five dimensions — definitions (30%), waterfall structure (25%), class configuration (20%), trigger logic (15%), and account structure (10%). Teaching models evolve automatically: each deal in a series inherits from its predecessor.

---

### Agent #4: Deal Amendment Agent
**What it does:** When a deal is amended (new servicing agreement, modified waterfall, changed thresholds), this agent processes the amendment, creates a versioned record of changes, and identifies which downstream artifacts need regeneration.

**Why it matters:** Deals live for 20-30 years. Amendments happen. Without version control:
- Analysts don't know if they're working with pre-amendment or post-amendment definitions
- Model updates risk breaking existing calculations
- There's no audit trail of what changed and when

This agent ensures amendments are tracked, originals are preserved, and the impact is clearly mapped.

**How it works:** Maintains a version chain (v1_original → v2_amended → active). Detects which sections changed, triggers re-extraction on those sections only, and flags cascade impacts (e.g., "this amendment changed 'OC Target' — your governing docs and payment model need regeneration").

---

### Agent #5: Payment Model Creation Agent
**What it does:** The core of the system. Reads governing documents and data CSVs, and generates a complete Python payment model — constants, rate calculations, interest waterfall, triggers, principal distribution, overcollateralization, and excess cashflow — with every formula citing its source.

**Why it matters:** This is the AI analyst replacement. Instead of a human manually translating 300+ pages of legal text into Excel formulas over days, the AI generates a complete, validated Python model. The model is:
- **Auditable** — every line cites the governing doc section
- **Testable** — runs against teaching output with $0.01 tolerance
- **Reproducible** — same inputs always produce same outputs
- **Version-controlled** — tracked in the deal folder

**How it works:** Reads all governing docs and CSV data, plans the model structure, generates code section-by-section (not monolithically), then self-validates against teaching output. If validation fails, it reads the diff report, fixes the issue, and re-validates — up to 3 self-healing iterations.

---

### Agent #6: Excel Payment Model Agent (Day 2)
**What it does:** Generates an Excel version of the payment model using openpyxl, with formulas that replicate the Python model's logic.

**Why it matters:** Analysts who don't know Python still need to review and understand the model. Excel is the industry's lingua franca for payment models. Cross-validation between Python and Excel provides a second layer of confidence.

**Status:** Parked for Day 2 implementation. The interface is designed for compatibility with the current system, ready to build when the core platform is operational.

---

### Agent #7: Model Auditor Agent
**What it does:** After a payment model is generated, the Auditor independently reviews it against the governing documents — checking that every constant has a source, every formula matches the legal text, and no values are hardcoded without citation.

**Why it matters:** The generation agent and auditor agent are deliberately separate — like having the model builder and the checker be different people. This separation of concerns is how the business works today (two independent Excel models), and we preserve that discipline in the AI system.

**How it works:** Reads the generated model line by line. For each constant, looks up the governing doc to verify the value. For each formula, compares mathematical representation to the legal text. Produces an audit report with pass/fail per check and full citations.

---

### Agent #8: Stress Testing Agent
**What it does:** Designs and executes stress scenarios that are aware of the deal's governing documents — not generic scenarios, but ones specifically calibrated to the deal's actual trigger thresholds.

**Why it matters:** Generic stress testing ("run 10% CDR") tells you what happens but doesn't tell you why it matters for this specific deal. This agent reads the deal's triggers (e.g., "delinquency event at 31.75% of current enhancement") and designs boundary scenarios that specifically test those transitions. It tells you: "At CDR 12.3%, the delinquency trigger fires in month 14, and Class M-8 through M-10 start taking losses."

**How it works:** Pre-built scenario library (high CDR, rapid prepay, rate spike) + deal-specific scenario design based on governing doc trigger thresholds. Executes scenarios through the cashflow engine and reports which triggers fire and which classes are affected.

---

### Agent #9: Regression Testing Agent
**What it does:** Captures "known good" model outputs as baselines, then detects any drift after model regeneration, amendment processing, or configuration changes.

**Why it matters:** When a model is updated — because of an amendment, a bug fix, or a governing doc correction — you need to know exactly what changed in the outputs. Did the fix actually fix the problem? Did it break something else? This agent provides that safety net.

**How it works:** Before any change, captures baseline outputs. After the change, re-runs the model and compares every number in every class to the baseline. Any deviation is flagged with the exact delta.

---

### Agent #10: Cash Flow Projection Agent + Tax Sub-Agent
**What it does:** Runs the payment model across the full remaining life of a deal under various assumptions (prepayment speeds, default rates, severity rates), producing multi-month projection curves.

**Why it matters:**
- **For investment decisions:** Project expected returns under different scenarios
- **For tax compliance:** The tax team uses cash flow projections for OID (Original Issue Discount) calculations and 8-K/10-K filings. Today this requires manual projection runs — the Tax Sub-Agent automates the output format they need.

**How it works:** A shared "cashflow engine" skill runs the payment model iteratively across N months, injecting scenario parameters at each step. The Tax Sub-Agent consumes the same projections but formats outputs for OID/tax reporting.

---

### Agent #11: Q&A / Research Agent
**What it does:** Answers natural language questions about any deal using the deal's structured knowledge — vector search, knowledge graph, extracted definitions, governing docs, and CSV data. Every answer comes with source citations.

**Why it matters:** Instead of opening a 300-page PSA and searching for "overcollateralization target," you type the question and get the answer with the exact section reference. This is valuable for:
- **Analysts** verifying model assumptions
- **Relationship managers** fielding client questions
- **Legal/compliance** checking specific provisions
- **New team members** ramping up on unfamiliar deals

**How it works:** Semantic search across the deal's vector store, knowledge graph traversal for term dependencies, and structured extraction lookups. Every answer cites its source file and section. If the information isn't in the deal's data, it says so — it never makes things up.

---

### Agent #12: Deal Lifecycle Monitor Agent
**What it does:** Ongoing surveillance of deal health. Tracks trigger status month-over-month, monitors approaching clean-up/call dates, and alerts when deals transition between states (e.g., pre-stepdown to stepdown).

**Why it matters:** Deals live for decades. Trigger events change payment allocation logic, clean-up calls affect deal wind-down, and stepdowns alter principal distribution priorities. Missing these transitions means paying investors incorrectly. This agent provides automated surveillance across the entire portfolio.

**How it works:** Reads monthly model outputs and compares against governing doc trigger conditions. Generates alerts with severity levels and recommended actions ("Delinquency Event tripped in Month 14 — verify principal distribution switch to sequential").

---

### Agent #13: Investor Reporting Agent
**What it does:** Generates investor reports from payment model outputs — class-level payments, trigger status, OC levels, CE calculations, and distribution amounts.

**Why it matters:** Investor reporting is a regulatory and contractual obligation. This agent automates the extraction and formatting of payment-model-derived data into standard report formats (Excel, PDF).

**Scope boundary:** This agent reports ONLY on data produced by the payment model. It does NOT handle collateral-level reporting (loan-level performance, delinquency breakdowns), which is handled by the existing mature collateral processing software that has been in production for 20-30 years.

**How it works:** Maps reporting requirements from the governing documents to available model outputs. Where requirements can be fulfilled, it assembles the data and generates formatted reports. Where requirements fall outside the model's scope (e.g., loan-level data), it clearly identifies the gap.

---

## Built-In Quality Assurance

This system doesn't just generate outputs — it continuously validates them. Three layers of quality control are embedded into every agent:

**Self-Healing Models:** Every agent output passes through a 5-dimension quality gate (completeness, accuracy, citation fidelity, structural conformance, deal scope compliance). If any dimension scores below threshold, the agent automatically retries with specific feedback — up to 3 times before escalating to a human reviewer. This means the system catches and fixes its own errors before anyone sees them.

**Confidence-Based Triage:** Not all AI outputs are created equal. The system assigns a confidence score to every extraction and generation. High confidence (≥90%) proceeds automatically. Medium confidence (66-89%) gets flagged for human review. Low confidence (<66%) halts processing entirely and generates a structured escalation report explaining exactly what went wrong, what was tried, and what needs human judgment. This prevents silent errors from reaching production.

**Traceability from Model to Source:** Every constant, formula, and payment rule in the generated Python model carries a standardized ID (DEF-001, RULE-001, etc.) that traces directly back to the governing document section it came from. The Model Auditor Agent independently verifies every one of these citations. If a formula can't be traced to a legal source — it doesn't ship.

**Pre-Mortem Risk Analysis:** Before generating a payment model, the system runs an adversarial analysis: "What could go wrong? Which definitions are ambiguous? Which waterfall rules have unusual conditions?" This primes the generation agent with awareness of deal-specific risks, catching edge cases that would typically surface only during manual QA.

---

## Why This Architecture?

### Agents, Not Monoliths

We could have built one giant application that does everything. Here's why we didn't:

| Monolith Approach | Agent Approach |
|-------------------|----------------|
| One failure breaks everything | Each agent operates independently — a Q&A outage doesn't affect model generation |
| Changing one feature risks breaking others | Each agent has clear boundaries and contracts |
| Hard to test ("does the whole system work?") | Each agent is independently testable with known inputs/outputs |
| Hard to extend ("add cash flow projections to the monolith") | Add a new agent with its own tools — the foundation supports it without changes |
| One permission model for everything | Each agent gets only the tools it needs (allowlist) — comparison agent gets read-only cross-deal; Q&A agent gets read-only deal-scoped |

### Separation of Concerns Mirrors the Business

The agent structure mirrors how the business actually works:

| Business Role | AI Agent |
|--------------|----------|
| Junior analyst reading the PSA | Ingestion Pipeline (#2) |
| Senior analyst building the model | Payment Model Creation (#5) |
| Checker verifying the model | Model Auditor (#7) |
| Risk analyst running stress tests | Stress Testing (#8) |
| Portfolio manager comparing deals | Document Comparison (#3) |
| Surveillance team monitoring triggers | Lifecycle Monitor (#12) |
| Reporting analyst generating reports | Investor Reporting (#13) |

---

## Key Technology Decisions

### Decision 1: Zero API Cost Beyond Copilot Subscription

**What:** All LLM tasks (document analysis, model generation, Q&A, comparison) run through VS Code GitHub Copilot Chat with Claude. No separate OpenAI/Anthropic API keys required.

**Why it matters:**
- **No incremental cost** — we already pay for GitHub Copilot. Every model generation, every Q&A query, every stress test analysis costs $0 in API fees.
- **No API key management** — no secrets to rotate, no billing to monitor, no rate limits to hit.
- **No vendor lock-in** — Copilot supports multiple models. If a better model appears, we switch without code changes.

**Trade-off acknowledged:** We're limited to what Copilot Chat can handle in a single conversation. For very long documents or complex multi-step generation, we use section-by-section processing rather than whole-document-at-once.

### Decision 2: Local-Only, Pip-Installable Stack

**What:** Every technology choice — Chroma DB, NetworkX, sentence-transformers, pdfplumber — is pip-installable and runs locally. No cloud services, no Docker, no external databases.

**Why it matters:**
- **Zero infrastructure cost** — no cloud instances, no database servers, no storage fees
- **Instant setup** — `pip install -r requirements.txt` and you're ready
- **Data stays local** — sensitive deal documents never leave the workstation
- **Offline capable** — works without internet (except for Copilot Chat LLM calls)

### Decision 3: Deal Isolation by Construction

**What:** Every operation is scoped to a single deal via the `DealScope` enforcer. Agents cannot accidentally access or contaminate other deals' data.

**Why it matters:** With 10,000+ deals, cross-contamination is not a theoretical risk — it's an eventual certainty if not prevented by design. One wrong vector query returning definitions from the wrong deal could produce a model with incorrect thresholds. We prevent this structurally:
- Separate Chroma collection per deal
- Path enforcement (can't read files outside the deal folder)
- LLM context boundary ("you are working on deal X, refuse requests about other deals")
- The one exception (Comparison Agent) is explicitly read-only

### Decision 4: Regex-First with LLM Fallback for Section Splitting

**What:** Documents are split into named sections using regex pattern matching (ARTICLE, Section, EXHIBIT patterns) with per-issuer configuration. When regex fails, Copilot Chat classifies the section.

**Why it matters:** With 500+ issuers and documents from 1990 to 2026, formatting varies enormously. Regex handles 90%+ of known shelf formats at zero cost and instant speed. The LLM fallback (via Copilot Chat, so still $0 cost) catches the remaining edge cases. This hybrid approach gives us both speed and coverage.

### Decision 5: Teaching Model Evolution

**What:** The first deal in a series gets a manually validated teaching model. Every subsequent deal in that series uses the previous deal's validated model as its starting point.

**Why it matters:** In practice, deals within a series (WHFSC 2024-1, 2024-2, 2024-3) are 90-96% identical. Starting from scratch for each one wastes time. By automatically identifying the closest match and carrying forward the validated model, generation becomes incremental — "update 3 constants and add 1 new account" instead of "build from zero."

---

## Cost Analysis

| Category | Current Cost | ABS Waterfall AI Cost | Savings |
|----------|-------------|----------------------|---------|
| **LLM / API** | N/A (manual process) | **$0** — included in Copilot subscription | No new cost |
| **Infrastructure** | N/A | **$0** — all local, pip-installable | No new cost |
| **Embedding Model** | N/A | **$0** — open-source all-MiniLM-L6-v2 | No new cost |
| **Vector Database** | N/A | **$0** — Chroma DB (pip install) | No new cost |
| **Graph Database** | N/A | **$0** — NetworkX (pip install) | No new cost |
| **Analyst Time** | 2 analysts × N days per deal | 1 analyst + AI × fraction of time | **~50% reduction in modeling labor** |
| **Error Detection** | Manual cross-check between 2 Excel models | Automated validation + audit + regression | **Faster detection, fewer misses** |

**Total incremental technology cost: $0 beyond existing Copilot subscription.**

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| **AI generates incorrect model** | Validation against teaching output within $0.01 per class. Model Auditor independently verifies every formula. Regression testing catches drift. Human reviewer always involved. |
| **Deal scoping violation** | DealScope enforcer structurally prevents cross-deal access. Unit-tested path escape detection. Separate vector collections per deal. |
| **Duplicate documents** | 3-layer detection (exact hash, near-duplicate, content overlap) blocks duplicates at ingestion. |
| **LLM hallucination** | "No source, no answer" policy. Every value must cite a governing doc section. Vector search and graph queries provide grounded context. |
| **Scale issues at 10K deals** | Per-deal Chroma collections isolate concerns. Portfolio-wide operations (comparison) are read-only and paginated. |
| **Amendment breaks model** | Version chain preserves originals. Cascade detection identifies what needs regeneration. Regression testing validates post-amendment outputs. |

---

## What Success Looks Like

**Month 1-2:** Foundation + ingestion pipeline operational. Drop a PSA.docx, get a fully structured deal with all five artifact groups.

**Month 2-3:** Payment model generation working. AI produces a Python model that matches the hand-built teaching model within $0.01 across all certificate classes.

**Month 3-4:** Full agent ecosystem live. Comparison agent finding closest deals, stress testing running boundary scenarios, Q&A answering questions with citations, regression testing catching drift, investor reports generating from model outputs.

**Ongoing:** Every new deal ingestion gets faster as the portfolio grows. The comparison agent has more data to match against. Teaching models evolve automatically within deal series. Portfolio-wide intelligence improves with scale.

---

## Summary

ABS Waterfall AI is a 13-agent AI ecosystem that transforms how we build, validate, and monitor ABS payment models. It replaces manual, error-prone document interpretation with automated, auditable, citation-backed model generation — at zero incremental technology cost. Every agent mirrors a real business function, every formula traces to its legal source, and every deal is isolated by construction. The system gets smarter with every deal we add to the portfolio.
