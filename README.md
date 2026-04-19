# context-pm  
**A local-first system for managing evolving product context at scale**

Built from real product work where documents, architecture decisions, test plans, and compliance requirements change daily—and Product Managers are expected to stay perfectly aligned across engineering, UX, leadership, and external stakeholders.

This repository documents **how to build a production-grade local knowledge system** using Retrieval Augmented Generation (RAG), context compaction, and agent-based workflows—designed specifically for PM realities, not demos.

> This is not a tutorial project.  
> It is a system that evolved through real usage, failed assumptions, and benchmarks.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![Claude API](https://img.shields.io/badge/Anthropic-Claude%20API-orange.svg)](https://www.anthropic.com/api)
---

## Table of Contents
1. #the-problem
2. What I Built
3. #architecture-overview
4. Implementation Guide
5. #performance-benchmarks
6. Lessons Learned
7. #when-to-use-which-architecture
8. #getting-started
9. #contributing

---

## The Problem

Product Managers don’t struggle with lack of information.  
We struggle with **too much of it—spread everywhere**.

In a typical multi‑project setup:
- Architecture docs in GitHub change multiple times a day
- Test plans evolve continuously as engineering iterates
- Compliance requirements arrive late and urgently
- Jira tickets, Confluence pages, and wikis diverge
- The same information must be rewritten for engineers, leadership, UX, and external partners

Answering a single question often means:
**search → read → reconcile → rewrite → reformat → resend**

This work is invisible, repetitive, and cognitively expensive.  
It is not product thinking. It is **context tax**.

The core problem is not access to documents.  
It is **maintaining an evolving, reliable understanding of the system** while everything changes underneath.

---

## What I Built

A **local-first knowledge and context system** designed around how PMs actually work.

At a high level, the system:
- Continuously reads multiple knowledge sources (GitHub, Confluence, Jira, wikis, documents)
- Converts raw content into structured markdown
- Tracks what changed instead of reprocessing everything
- Retrieves only relevant context per query (RAG)
- Adapts responses to different stakeholder audiences
- Optimizes for cost and latency through context compaction
- Supports both single-agent and multi-agent workflows

Initially, this was:
- simple scripts  
then:
- Python functions generated via AI  
then:
- a local CLI chatbot  
then:
- a browser-based UI  
then:
- multi-agent orchestration experiments  

The system evolved because real usage demanded it.

---

## Architecture Overview


### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      USER INTERFACE                          │
│              (CLI / Web App / API)                           │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
┌─────────▼──────────┐       ┌─────────▼──────────┐
│  Single-Agent      │       │  Multi-Agent        │
│  Architecture      │       │  Architecture       │
│  (Monolithic)      │       │  (Orchestrated)     │
└─────────┬──────────┘       └─────────┬──────────┘
          │                             │
          │                   ┌─────────▼──────────┐
          │                   │   Orchestrator     │
          │                   │  (Intent Routing)  │
          │                   └─────────┬──────────┘
          │                             │
          │         ┌───────────────────┼───────────────────┐
          │         │                   │                   │
          │    ┌────▼─────┐   ┌────────▼────┐   ┌─────────▼──────┐
          │    │ Research │   │  Content    │   │   Technical    │
          │    │  Agent   │   │ Generator   │   │    Advisor     │
          │    └────┬─────┘   └────────┬────┘   └─────────┬──────┘
          │         │                  │                   │
          └─────────┼──────────────────┼───────────────────┘
                    │
         ┌──────────▼──────────────────────────────────────┐
         │         Document Retrieval Layer                 │
         │  - Semantic Search (Embeddings)                  │
         │  - Keyword Search (BM25)                         │
         │  - Hybrid Re-ranking                             │
         └──────────┬─────────────────────────────────────┘
                    │
         ┌──────────▼─────────────────────────────────────┐
         │          Data Integration Layer                  │
         │  ┌──────────┬──────────┬──────────┬──────────┐ │
         │  │ GitHub   │Confluence│  Jira    │  Office  │ │
         │  │ (OAuth2) │ (Bearer) │ (OAuth)  │  Docs    │ │
         │  └──────────┴──────────┴──────────┴──────────┘ │
         └────────────────────────────────────────────────┘
                    │
         ┌──────────▼─────────────────────────────────────┐
         │       Incremental Sync Engine                   │
         │  - SHA-256 Hash Change Detection                │
         │  - State Persistence (JSON)                     │
         │  - Atomic Writes & Error Recovery               │
         └─────────────────────────────────────────────────┘
```

At its core, the system has four layers:

### 1. Ingestion & Normalization
- Pulls content from:
  - Git repositories
  - Confluence / wiki systems
  - Jira-like issue trackers
  - Local Word, PowerPoint, and Excel files
- Converts all content into **clean markdown**
- Removes formatting noise while preserving meaning

### 2. Change Detection & State
- Each document is hashed (content-level)
- Only modified content is reprocessed
- Prevents unnecessary re-embedding
- Preserves historical decisions explicitly

### 3. Retrieval (RAG Layer)
Yes—this is Retrieval Augmented Generation, but opinionated.

- Semantic retrieval + lightweight keyword filtering
- Decision-focused chunking (not arbitrary token splits)
- Context compaction before model calls
- Audience-aware rendering

RAG here is not “search and paste”.  
It is **retrieve → reason → reframe**.

### 4. Reasoning Layer
Two supported modes:
- **Single-agent (monolithic)** for speed and analysis
- **Multi-agent (orchestrated)** for specialization and writing tasks

A feature flag allows switching and benchmarking both.

---

## Implementation Guide

### Prerequisites

- Python 3.8+
- Anthropic API key (Claude models)
- Access to enterprise data sources (GitHub, Confluence, etc.)
- Basic understanding of LLM APIs and vector search

### Quick Start

```bash
# Clone repository
git clone https://github.com/yourusername/production-ai-agent.git
cd production-ai-agent

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your API credentials

# Run initial sync
python src/main.py sync

# Start querying
python src/main.py query "Find all documents about X"
```

### Project Structure

```
production-ai-agent/
├── src/
│   ├── agents/
│   │   ├── single_agent.py       # Monolithic agent
│   │   ├── orchestrator.py       # Intent classification
│   │   └── specialists.py        # Specialist agents
│   ├── data_integration/
│   │   ├── adapters/             # Data source adapters
│   │   └── sync_engine.py        # Incremental sync
│   ├── retrieval/
│   │   ├── hybrid_search.py      # Semantic + keyword search
│   │   └── context_optimizer.py  # Token budget management
│   └── api/
│       └── app.py                # Web interface
├── config/
│   ├── settings.py
│   └── config.json
├── data/
│   ├── documents/                # Cached documents
│   ├── embeddings/               # Vector embeddings
│   └── sync_state.json           # Sync state tracking
├── benchmark/
│   ├── benchmark.py              # Performance testing
│   └── results/                  # Benchmark outputs
├── tests/
├── requirements.txt
└── README.md
```

### Step-by-Step Implementation

**Phase 1: Foundation (Single-Agent)**
1. Set up data integration adapters
2. Implement incremental sync engine
3. Build hybrid retrieval system
4. Create single-agent handler
5. Add monitoring and cost tracking

**Phase 2: Multi-Agent Evolution**
1. Implement orchestrator for intent classification
2. Create specialized agents (Research, Content, Technical, Business)
3. Add feature flag system (single/multi/compare modes)
4. Build fallback mechanisms

**Phase 3: Optimization**
1. Add agent pooling (eliminate cold start)
2. Remove two-stage processing bottleneck
3. Implement caching layer
4. Add hybrid routing (task-based architecture selection)

**Full implementation guide with code examples:** See [IMPLEMENTATION.md](docs/IMPLEMENTATION.md)

## Performance Benchmarks

### Test Methodology

**Test Setup:**
- 10 real-world queries across all agent types
- 2 architectures (single-agent vs multi-agent)
- Metrics: Response time, token usage, cost per query, output quality
- Same API endpoint, no caching between runs

**Query Types:**
1. Research / information lookup
2. Email writing
3. Technical feasibility assessment
4. Scope clarification
5. Technical Q&A
6. Presentation creation
7. Risk assessment
8. Architecture documentation
9. Requirements definition
10. Integration analysis

---

### Overall Results

| Metric | Single-Agent | Multi-Agent | Difference | Analysis |
|--------|-------------|-------------|------------|----------|
| **Total Time** | 583s (9.7 min) | 801s (13.4 min) | **+37% slower** ❌ | Multi-agent slower overall |
| **Avg per Query** | 58.3s | 80.1s | **+37% slower** ❌ | Consistent overhead |
| **Total Tokens** | ~120K | ~380K | **+217% more** ❌ | 3x token multiplication |
| **Cost per Query** | ~$0.90 | ~$0.30 | **-67% cheaper** ✅ | Smaller models offset token increase |

---

### Performance by Task Type

| Task | Single-Agent | Multi-Agent | Speed Change | Winner | Why? |
|------|-------------|-------------|--------------|---------|------|
| **Email Writing** | 30s | 29s | +3% faster | Multi ✅ | Single model call, specialized prompt |
| **Presentation** | 57s | 44s | **+23% faster** | Multi ✅ | Template-based, no research overhead |
| **Architecture Docs** | 81s | 58s | **+28% faster** | Multi ✅ | Focused specialist, no context switching |
| **Research (simple)** | 23s | 103s | **-346% slower** | Single ✅ | Cold start + orchestration overhead |
| **Technical Analysis** | 79s | 119s | -51% slower | Single ✅ | Two-stage processing penalty |
| **Scope Clarification** | 53s | 94s | -79% slower | Single ✅ | Orchestration + context duplication |

**Key Finding:** Multi-agent is faster for **creative/writing tasks**, slower for **analytical/research tasks**.

---

### Root Cause Analysis

**Why was multi-agent slower overall?**

**1. Cold Start Penalty (-346% on first query)**
- All 5 specialist agents initialize upfront (~70+ seconds)
- Single-agent has no initialization overhead
- Real users restart sessions frequently (warm-start testing was misleading)

**2. Token Multiplication (+217% more tokens)**
- Single-agent: 1 API call with full context (~15K tokens)
- Multi-agent: 3 sequential API calls (Orchestrator → Research → Specialist ~45K tokens)
- Context gets duplicated across agent stages

**3. Orchestrator Overhead (Every query)**
- 1-3 seconds per query for classification
- 500-1000 tokens per query for routing
- Cumulative overhead that produces no user value

**4. Two-Stage Processing**
- Research Agent retrieves documents
- Specialist Agent processes with documents
- Double the API latency even with smaller models

---

### Where Multi-Agent Excelled

**Creative/Writing Tasks (+15-28% faster):**
- Content Generator consistently outperformed
- Specialized prompts tuned for writing
- Single model call (no research pre-fetch)
- Better quality: more concise, professional tone, clearer structure

**Cost Efficiency (-67%):**
- Haiku for orchestration ($0.00000025/token input)
- Sonnet for most specialists ($0.000003/token input)
- Opus only for complex analysis ($0.000015/token input)
- Smaller models offset 3x token increase

**Quality Improvements:**
- Structured responses (clearer sections)
- Appropriate depth (presentations concise, analysis thorough)
- Consistent formatting (predictable output style)


---

## Lessons Learned

### 1. Raw Documents Kill RAG Quality
Formatting noise destroys relevance. Markdown normalization mattered more than embeddings.

### 2. Incremental RAG Beats “Just Reindex”
Change detection reduced cost and latency more than model changes.

### 3. Smaller Context > Bigger Models
Clean context + smaller models repeatedly outperformed messy context + large models.

### 4. Elegant Architecture Can Benchmark Poorly
Some designs felt right and performed badly. Measurement changed everything.

### 5. PM Knowledge Is Structural
Decisions, trade-offs, and evolution matter more than raw text.

### 6. **What I calculated:**
- Haiku costs 10x less per token than Opus → TRUE
- Sonnet costs 5x less per token than Opus → TRUE

**What I missed:**
- Making 3 API calls with 3x more tokens meant total cost was similar, just distributed differently
- Savings came from model selection, not architectural efficiency

**The lesson:** Optimize total tokens and API calls, not just model selection.

---

## When to Use Which Architecture

### Use Single-Agent When:
- Low latency matters
- Queries are analytical or investigative
- Context scope is narrow
- You want architectural simplicity

### Use Multi-Agent When:
- Outputs are creative or communicative
- Stakeholder-specific formatting matters
- Cost per query is critical
- Latency trade-offs are acceptable

### Best Practice
Start simple. Add agents only where specialization clearly pays off.

---

## Getting Started

### Prerequisites
- Python 3.9+
- Local LLM API access (cloud or local, configurable)
- Access credentials for your content sources

### Basic Flow
1. Configure data sources
2. Run initial ingestion
3. Validate markdown outputs
4. Enable retrieval
5. Query via CLI or browser UI
6. Benchmark before scaling

> This repository focuses on **patterns and architecture**.  
> Source adapters and credentials are intentionally abstracted.

--- Details 

## Contributing

Contributions are welcome—especially around:
- additional document extractors
- improved context compaction strategies
- benchmarking methodologies
- visualization and UX improvements

Please:
- keep changes generic
- avoid proprietary integrations
- document trade-offs honestly

---

### Final Thought

Product work is about managing **evolving truth**.

Treating context as a first-class system—rather than a side effect of documentation—changed how this work scaled.

If this helps you reclaim time, clarity, or sanity:  
that’s success.

s.

---



## 📄 License

MIT License - see [LICENSE](LICENSE) file for details.

---

## 📞 Contact & Connect

**Author:** Vinodh.L.K - Product Manager at SAP working on AI-driven payroll automation

**LinkedIn:** [Connect with me](https://www.linkedin.com/in/lkvinodh/)


**Read the full story:** 
- [Building Production-Grade AI Agents (LinkedIn Article)](#)
- [From One Agent to Many (LinkedIn Article)](#)
- [Performance Benchmarking & Lessons (LinkedIn Article)](#)

---

## 🎓 Additional Resources

**Documentation:**
- [Full Implementation Guide](docs/IMPLEMENTATION.md)
- [Architecture Deep Dive](docs/ARCHITECTURE.md)
- [Benchmarking Methodology](docs/BENCHMARKS.md)
- [Production Deployment Guide](docs/DEPLOYMENT.md)

**Related Projects:**
- [Anthropic Claude API Documentation](https://docs.anthropic.com/)
- [LangChain Framework](https://python.langchain.com/)
- [Sentence Transformers](https://www.sbert.net/)

---

## 🙏 Acknowledgments

- **Anthropic** for Claude API (Haiku, Sonnet, Opus models)
- **SAP** for the professional context and real-world use cases that inspired this work
- **Open-source community** for foundational tools and frameworks

---

## ⚠️ Disclaimer

*This project uses publicly available tools and is designed for local deployment—no proprietary SAP information was shared with external services. Built as a productivity tool while maintaining full compliance with corporate data governance policies.*

*Views and opinions expressed are my own and do not represent those of SAP SE or its affiliates.*

---

**Build systems, not demos. Measure everything. Iterate based on data. 🚀**

---



*Last Updated: April 2026*
