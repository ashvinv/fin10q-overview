# Fin10Q

**Production-grade RAG platform for SEC filing intelligence and quantitative signal extraction.**

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=white)](https://react.dev/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Claude API](https://img.shields.io/badge/Claude-Sonnet%204%20%2B%20Haiku%203.5-D97757?logo=anthropic&logoColor=white)](https://www.anthropic.com/)
[![Voyage AI](https://img.shields.io/badge/Embeddings-Voyage%20Finance%202-7C3AED)](https://www.voyageai.com/)
[![ChromaDB](https://img.shields.io/badge/Vector%20DB-ChromaDB-FF6B6B)](https://www.trychroma.com/)
[![PostgreSQL](https://img.shields.io/badge/Postgres-15-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)

---

## Overview

Fin10Q ingests S&P 500 10-K and 10-Q filings directly from SEC EDGAR, parses their structured iXBRL into a domain-specific knowledge layer, and surfaces both natural-language answers and quantitative trading signals. It combines retrieval-augmented generation with a denormalized factor panel, so analysts can ask "What drove Tesla's stock-based compensation spike in FY2025?" and quantitative researchers can screen 33 companies by sentiment delta or run an event study on filing-driven returns — from the same underlying corpus.

The system was built to address a specific gap: most financial RAG demos read PDFs, embed everything uniformly, and call it done. Real SEC filings have predictable structure (Items 1-15), embedded structured data (iXBRL), and produce different question patterns (metric retrieval vs. footnote investigation vs. cross-sectional comparison). Fin10Q treats those differences as architectural inputs rather than edge cases.

---

## Architecture

> Architecture diagram coming soon. See `Financial RAG System Guide` for the high-level data flow across ingestion, retrieval, and signal layers.

The system runs as 8 Docker services: FastAPI backend, three ARQ workers, React frontend, PostgreSQL, Redis, and ChromaDB. The ingestion pipeline is a 9-step process executed asynchronously per filing:

```
EDGAR Download -> iXBRL Enrichment -> HTML to Markdown -> Section ID
              -> XBRL Fact Extraction -> Table Summarization
              -> Hierarchical Chunking -> Embedding -> Storage
```

Two databases sit behind the API. ChromaDB holds 89,546 chunk-level embeddings with rich metadata (ticker, section, period, chunk type). PostgreSQL holds extracted XBRL facts, signal time series, the factor panel, and job state. Queries route to one, the other, or both depending on the question's nature.

---

## Key Capabilities

### Natural language querying with intelligent routing
Ask `What was Apple's stock-based compensation in FY2024?` and the system routes the query through a three-path classifier before any retrieval happens. Metric queries with structured XBRL data bypass the vector database entirely. Qualitative footnote queries (SBC, leases, goodwill, contingencies — 90 trigger keywords) go straight to RAG. Temporal questions ("revenue trend over the last 3 years") use a hybrid: SQL fetches the structured time series, RAG retrieves narrative context, and a final LLM call composes the answer.

### Query decomposition and multi-query retrieval
Complex questions are decomposed by Claude Haiku into 3-5 targeted sub-queries before retrieval. Each sub-query runs against ChromaDB independently, results are deduplicated by document hash and re-ranked by distance score. This dramatically improves recall on questions that span multiple filings or sections.

### Structured signal extraction (5 signal types)
Every ingested filing produces up to 9 signal records covering MD&A, Risk Factors, Financial Statements, and Legal Proceedings. Two signals are LLM-extracted (sentiment, guidance, risk_change), two are computed at zero API cost from already-stored ChromaDB vectors and word-frequency analysis (delta, uncertainty), and the categorical signals capture forward-looking management posture.

### Cross-sectional screener
Filter and rank the universe by any signal dimension across 33 companies. Supports inequality and range operators, sector grouping, period selection (latest filing, trailing N quarters, or specific fiscal year), and returns percentile ranks alongside raw values. Sub-second response times across the 760-row factor panel.

### Event study engine
Define an event from any signal threshold (e.g., `delta_mda > 0.25`), and the engine identifies all matching filings, pulls daily returns from the price database around each event date, computes cumulative abnormal returns at 1/5/21/63 trading-day windows, benchmarks against an equal-weight or sector-matched control group, and runs one-sample t-tests on mean CAR.

### Factor panel export
A denormalized signal matrix — one row per filing, signals pivoted into columns — exportable as CSV or JSON for direct loading into pandas, R, or any quant workflow. Excludes filings flagged as unhealthy by the verification layer.

### Universe health dashboard
A per-company filing coverage matrix tracking healthy / warning / failed flags. The verification layer runs four automated checks per filing: chunk count sanity, required section presence, embedding count consistency with ChromaDB, and content quality.

---

## Tech Stack

| Layer | Component | Purpose |
|-------|-----------|---------|
| **Data** | SEC EDGAR API | Source for 10-K and 10-Q filings, iXBRL data |
| | Yahoo Finance | Daily price history (primary) |
| | Tiingo | Delisted ticker price history |
| **Parsing** | Custom HTML to Markdown | Bespoke converter with 5-phase heading detection |
| | iXBRL Enricher | Inline tag extraction before markdown conversion |
| | Section Identifier | TOC filtering, ATX detection, cross-reference reassignment |
| **Storage** | PostgreSQL 15 | Structured data: companies, filings, signals, factor panel, prices |
| | ChromaDB 0.5.5 | Vector store: 89,546 chunks with metadata filters |
| **Retrieval** | Voyage Finance 2 | Finance-domain embeddings (1024-dim) |
| | 3-path Query Router | SQL / RAG / hybrid classification |
| | Multi-query Retrieval | Sub-query decomposition, dedup, re-rank |
| **Generation** | Claude Sonnet 4 | Complex queries, multi-document synthesis |
| | Claude Haiku 3.5 | Routing, table summarization, signal extraction (~$0.004/section) |
| **Orchestration** | LangChain | Prompt templating, retry logic |
| | ARQ + Redis | Background job queue (3 workers, 21 concurrent slots) |
| **Backend** | FastAPI | REST API with async workers |
| **Frontend** | React 18 + Material UI | Analytics dashboard, screener, event study UI |
| **Charts** | Recharts | Quintile bar charts, CAR plots, time series |
| **Deployment** | Docker Compose | 8-service stack |
| **Evaluation** | RAGAS-inspired suite | 15 test cases scoring accuracy, completeness, route correctness |

---

## Architecture Decisions

**1. No PDF parsing.** Filings are pulled as HTML with iXBRL inline tags directly from EDGAR. PDFs lose the structured tags, the section markers, and the table semantics that make SEC filings actually parseable. Working from HTML preserves the document graph.

**2. Hierarchical chunking respecting filing structure.** Chunks aren't sliced at fixed token counts. The chunker walks the section tree (Item 1A, 7, 8, etc.) and produces chunks that respect paragraph and footnote boundaries. Item 8 footnotes get larger chunk windows (1024 tokens) than narrative MD&A (700 tokens) because their content density is different.

**3. Haiku-summarized tables before embedding.** Raw markdown tables embed poorly — the dense numeric content swamps the semantic signal. Each financial table is summarized in 2-3 sentences by Claude Haiku, the summary is embedded, and the raw table is preserved in metadata so the LLM can read it at answer time. This was the single largest retrieval quality improvement in the project.

**4. Three-path query router.** A query like "What was Apple's revenue in FY2024?" doesn't need a vector database — XBRL has the exact answer. A query like "What did Apple say about AI risk?" doesn't need SQL. A query like "How has revenue grown over the last 3 years and what's driving it?" needs both. The router classifies the question first and dispatches accordingly.

**5. Finance-domain embeddings.** Voyage Finance 2 is purpose-built on financial corpora and outperforms general-purpose embeddings on filing retrieval. The cost difference is negligible against the gain in retrieval quality.

**6. Multi-LLM cost optimization.** Sonnet 4 handles complex synthesis, Haiku 3.5 handles routing, table summarization, signal extraction, and query decomposition. Total LLM cost per ingested filing is roughly $0.07 versus $0.50+ if Sonnet did everything.

**7. Dual database architecture.** ChromaDB for similarity search, PostgreSQL for structured XBRL facts and time-series joins. The factor panel is a materialized denormalization that pivots `filing_signals` into a wide format optimized for cross-sectional queries — analysts get a clean ticker × date × factor matrix without paying the cost of a join at query time.

---

## Signal Types

| Signal | Range | Source | Cost | Description |
|--------|-------|--------|------|-------------|
| **Sentiment** | -1.0 to +1.0 | Claude Haiku | ~$0.004/section | Tone of the section text — growth, margin pressure, restructuring language |
| **Guidance** | categorical | Claude Haiku | shared | Forward-looking direction: raised / maintained / lowered / not_mentioned |
| **Risk Change** | categorical | Claude Haiku | shared | Risk posture shift: new_risks / escalated / de_escalated / stable |
| **Delta** | 0.0 to ~1.0 | ChromaDB embeddings | $0.00 | `1 − cosine(current_section, prior_section)` — quarter-over-quarter text change magnitude |
| **Uncertainty** | 0.0 to ~0.10 | Word counting | $0.00 | Loughran-McDonald hedge word density (~80 words: "may", "could", "estimate", "uncertain"...) |

The delta and uncertainty signals are zero marginal cost — delta uses embedding vectors that are already in ChromaDB from the ingestion pipeline, and uncertainty is pure word-frequency analysis. This means the cost of running every signal across the entire universe scales sublinearly with universe size.

---

## API Surface (selected endpoints)

```
POST   /query/                        Natural language query against filings
POST   /compare/                      Multi-company comparison
POST   /ingestion/bulk                Bulk ingest by ticker list or group_id
POST   /universe/build                Universe selection with cost estimate
GET    /universe/health               Per-company filing coverage matrix
GET    /factors/panel                 Export factor panel (CSV/JSON)
POST   /factors/rebuild               Rebuild factor panel from signals
POST   /factors/screen                Cross-sectional screener
POST   /factors/event-study           CAR computation with t-test statistics
POST   /signals/extract               Trigger signal extraction
POST   /signals/backtest              Signal-return correlation, quintile sorts
GET    /signals/{ticker}/tone-trend   Sentiment erosion monitor
POST   /eval/run                      Trigger evaluation suite
```

---

## Key Findings From the Data

The system has processed **760 filings across 33 S&P 500 companies spanning 6 years**, producing **89,546 chunks in ChromaDB** and **1,644 extracted signals**.

**Risk Factor sections are structurally more pessimistic than MD&A.** Average sentiment in Risk sections is **-0.23** versus **+0.18** in MD&A — the gap is consistent across sectors and reflects the disclosure convention rather than actual company-specific stress.

**Risk Factor language is materially more hedged.** Hedge word density in Risk sections averages **3.16%** versus **1.14%** in MD&A — roughly 3x. Useful for quants who want to factor-adjust uncertainty signals by section type.

**Meta pays roughly 10% of revenue in stock-based compensation, consistently.** The ratio has been stable across 4+ years of filings — a structural feature of their compensation model, not a one-off.

**Nvidia's SBC ratio compressed from ~10% of revenue to ~3% as AI revenue exploded ~8x.** A clean example of denominator-driven margin improvement that wouldn't show up in absolute SBC numbers.

**Alphabet paid $27.1B in SBC in FY2025** — extracted directly from the Item 8 footnotes via the financial statements RAG path.

These insights surfaced through the same screener and query infrastructure that's exposed via the API.

---

## Project Status

**Production / Active Development.** Core ingestion, retrieval, signal extraction, and analytics layers are functional. The platform currently runs against 33 companies (Top 25 dynamic group + MAG7 + NFLX). Expansion to the full S&P 500 is straightforward — the Universe Builder estimates ~$300-400 in embedding costs and ~$100 in signal extraction costs for the full ingestion.

Recent additions include a 90-keyword footnote query router, a cross-reference reassignment fix for filings where Item 8 is a one-line pointer to Item 15, and a tiered ingestion preset (Top 25 / Quant 50 / Quant 100) with smart period defaults.

---

## About the Author

**Ashvin Viswanathan, CFA**

Quantitative portfolio manager with 20 years of institutional investment experience, currently building AI-native financial intelligence systems.

- B.Math, Computer Science — University of Waterloo
- Master of Mathematical Finance — University of Toronto
- LinkedIn: [linkedin.com/in/ashvin-viswanathan-4690148](https://www.linkedin.com/in/ashvin-viswanathan-4690148)
- Co-author: *"Location Density, Systematic Risk, and Cap Rates: Evidence from REITs,"* Real Estate Economics, Vol. 50, No. 2, 2022

Fin10Q is a portfolio project demonstrating production RAG architecture, multi-LLM cost optimization, and the integration of unstructured filing text with structured XBRL data for quantitative research workflows.
