# Assignment 2: Production-Grade RAG Pipeline with Advanced Chunking & Multi-Tier Caching

## Problem Statement

In Assignment 1, you built a classic RAG pipeline with retrieval, reranking, and generation using Pinecone and OpenAI. While functional, that implementation treated chunking as a flat operation and performed a fresh retrieval + LLM call for every single query — regardless of whether the same (or a similar) question had already been answered.

In real-world production systems, this is unacceptable. Redundant LLM calls waste money, add latency, and don't scale. Naive fixed-size chunking loses document context and produces poor retrieval results.

**Your task in Assignment 2 is to evolve your RAG pipeline into a production-grade system** by solving two core problems:

### Problem 1 — Intelligent Document Chunking

Flat, fixed-size chunks lose the relationship between a paragraph and its parent section/document. A user asking a broad question gets a narrow fragment with no surrounding context.

You must implement:

1. **Parent-Child Chunking**: Split documents into large parent chunks and smaller child chunks. Retrieval happens at the child level (for precision), but the parent chunk is passed to the LLM (for context). This ensures the model always sees the bigger picture.

2. **One Additional Chunking Strategy of Your Choice**: Research and implement at least one more chunking approach. Examples include (but are not limited to):
   - Semantic chunking (split based on embedding similarity shifts)
   - Sliding window with overlap
   - Markdown/HTML-aware structural chunking
   - Recursive character chunking with metadata preservation
   - Agentic chunking (LLM-driven boundary detection)

   Justify your choice in the architecture document.

### Problem 2 — Three-Tier Caching System

Every query should not hit the vector database and LLM. You must implement a **three-tier caching layer** that sits between the user query and the retrieval/generation pipeline:

| Tier | Name | Behavior |
|------|------|----------|
| **Tier 1** | **Exact Cache** | If the incoming query exactly matches a previously seen query (after normalization), return the cached response immediately. No retrieval, no LLM call. |
| **Tier 2** | **Semantic Cache** | If no exact match is found, compute the query embedding and compare it against cached query embeddings using cosine similarity. If similarity exceeds a configurable threshold, return the cached response. No retrieval, no LLM call. |
| **Tier 3** | **Retrieval Cache** | If no semantic match is found, check if relevant chunks for this query (or a similar query) already exist in cache. If yes, skip the vector DB call and pass the cached chunks directly to the LLM for generation. Only the LLM call happens — retrieval is skipped. |

If all three tiers miss, fall through to the full pipeline: retrieval → reranking → generation, and cache the results at all three tiers for future queries.

---

## Project Guidelines

### Architecture & Code Quality

This is not a notebook exercise. Your submission must reflect **production-grade software engineering**:

- **Project Structure**: Follow a clean, modular folder structure with separation of concerns. No monolithic scripts. See the recommended structure below.
- **Configuration Management**: All tunable parameters (model names, chunk sizes, similarity thresholds, cache TTLs, Pinecone index names, etc.) must be managed through a `config.yaml` or `config.py` — not hardcoded.
- **Environment Variables**: API keys, secrets, and environment-specific values must live in a `.env` file and must **never** be committed to the repository. Provide a `.env.example` with placeholder values.
- **Separation of Concerns**: Chunking logic, caching logic, retrieval logic, reranking logic, and generation logic should live in separate modules. Each should be independently testable and replaceable.
- **Logging**: Implement proper logging (not print statements) to trace cache hits/misses, retrieval counts, and latency.
- **Error Handling**: Handle API failures, empty retrievals, and edge cases gracefully.

### Recommended Folder Structure

```
rag-project2/
├── README.md
├── .env.example
├── config.yaml
├── requirements.txt
├── main.py                     # Entry point
├── src/
│   ├── __init__.py
│   ├── chunking/
│   │   ├── __init__.py
│   │   ├── parent_child.py     # Parent-child chunking
│   │   └── <your_strategy>.py  # Your chosen chunking strategy
│   ├── caching/
│   │   ├── __init__.py
│   │   ├── exact_cache.py      # Tier 1 - Exact match
│   │   ├── semantic_cache.py   # Tier 2 - Semantic similarity
│   │   └── retrieval_cache.py  # Tier 3 - Chunk-level cache
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── retriever.py        # Pinecone retrieval logic
│   │   └── reranker.py         # Reranking logic
│   ├── generation/
│   │   ├── __init__.py
│   │   └── generator.py        # LLM generation logic
│   └── utils/
│       ├── __init__.py
│       ├── embeddings.py       # Embedding utilities
│       └── logger.py           # Logging configuration
├── data/                       # Input documents (do NOT commit large files)
│   └── .gitkeep
├── tests/                      # Unit/integration tests (optional but encouraged)
│   └── ...
└── docs/
    └── architecture.md         # Architecture document (REQUIRED)
```

You may adapt this structure, but the principles — modularity, separation, config-driven — must hold.

### Technology Stack

| Component | Requirement |
|-----------|-------------|
| Vector DB | Pinecone |
| Embeddings | OpenAI Embeddings API |
| LLM | OpenAI (GPT-4o / GPT-4o-mini or latest available) |
| Reranking | Your choice (Cohere Rerank, cross-encoder, or similar) |
| Caching Backend | Your choice (in-memory dict, Redis, SQLite — justify in architecture doc) |
| Language | Python 3.10+ |

---

## Deliverables

### 1. GitHub Repository

| Item | Details |
|------|---------|
| **Repository** | Public or private GitHub repo (add `admin@datasenseia.co.in` as a collaborator if private) |
| **Code** | Complete, runnable codebase following the guidelines above |
| **`.env.example`** | Template with all required environment variables (no real keys) |
| **`config.yaml`** | All configurable parameters with sensible defaults |
| **`requirements.txt`** | Pinned dependencies |
| **`README.md`** | Setup instructions, how to run, and brief description |

### 2. Architecture Document (`docs/architecture.md`)

This is a **required** deliverable. It must include:

1. **System Architecture Diagram** — A visual diagram showing the complete flow: query → cache tiers → retrieval → reranking → generation → response. Use any tool (draw.io, Excalidraw, Mermaid, hand-drawn scan — whatever communicates clearly).

2. **Chunking Strategy Explanation**
   - How your parent-child chunking works (parent size, child size, overlap, metadata linking)
   - What second chunking strategy you chose and why
   - Comparison: when would you use one vs the other?

3. **Caching Architecture**
   - How each of the three cache tiers is implemented
   - What data is stored at each tier
   - Cache invalidation strategy (TTL, LRU, manual — your choice, but justify it)
   - Configurable parameters (thresholds, TTLs, max cache size)

4. **Design Decisions & Trade-offs**
   - Why you chose your caching backend (Redis vs in-memory vs SQLite)
   - How you handle cache warming and cold starts
   - Any trade-offs you made and why

5. **Query Flow Walkthrough** — Trace a single query through your entire system step by step, showing what happens at each stage (with example cache hit/miss scenarios).

---

## Evaluation Criteria

| Criteria | Weight | What We Look For |
|----------|--------|------------------|
| **Parent-Child Chunking** | 20% | Correct parent-child relationship, metadata linking, parent context passed to LLM |
| **Second Chunking Strategy** | 10% | Thoughtful choice, working implementation, justified in architecture doc |
| **Three-Tier Caching** | 30% | All three tiers working correctly, proper fallthrough logic, configurable thresholds |
| **Code Quality** | 20% | Clean structure, separation of concerns, config-driven, proper logging, error handling |
| **Architecture Document** | 15% | Clear diagram, thorough explanations, well-reasoned design decisions |
| **Bonus** | 5% | Tests, cache analytics/metrics, CLI interface, Streamlit/Gradio demo, or any other thoughtful addition |

---

## Submission

| | |
|---|---|
| **Deadline** | **March 6, 2026** |
| **Submit to** | **admin@datasenseia.co.in** |
| **Email Subject** | `RAG Assignment 2 - <Your Full Name>` |
| **Email Body** | Include your GitHub repository link and any setup notes |

Late submissions will not be accepted unless prior approval is obtained.

---

## Getting Started

1. Clone this repository
2. Copy `.env.example` to `.env` and fill in your API keys
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Update `config.yaml` with your Pinecone index name and desired parameters
5. Place your documents in the `data/` directory
6. Run the pipeline:
   ```bash
   python main.py
   ```

---

## Important Notes

- **Do NOT commit API keys or secrets.** Use `.env` and add it to `.gitignore`.
- **Do NOT commit large data files.** Use `.gitkeep` in empty directories and document data sources.
- Your code should be runnable by someone who clones the repo, sets up `.env`, and follows your README.
- Focus on getting the core pipeline working end-to-end first, then optimize.
- Ask questions early — don't wait until the deadline.

---

*For questions or clarifications, reach out to admin@datasenseia.co.in*
