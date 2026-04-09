# Shopping Assistant Agent — Research Plan

## Overview

This document outlines a structured research plan for designing a conversational shopping assistant agent (e.g., Amazon Rufus, Walmart Sparky). The core focus is on **how the agent integrates with the underlying product search stack**, spanning query understanding, retrieval, ranking, and the agentic orchestration layer on top.

---

## 1. Landscape Survey

### 1.1 Industry Deployments
- **Amazon Rufus** — multimodal, conversational product discovery; integrated into search + PDP (product detail page)
- **Walmart Sparky** — generative AI shopping assistant; real-time catalog search + personalization
- **Google Shopping AI** — LLM-backed Shopping Graph; intent detection + product summaries
- **Perplexity Shopping** — retrieval-augmented product cards with buy buttons
- **Meta AI Shopping** — social-graph-aware recommendations within Instagram/WhatsApp

### 1.2 Key Research Questions
1. How does the agent translate a natural-language shopping request into search engine queries?
2. How are multi-turn conversational context and constraints (budget, brand, size, etc.) maintained?
3. How is retrieved product data re-ranked and synthesized into a coherent assistant response?
4. Where does the agent call search vs. generate vs. retrieve from memory?
5. What latency / cost tradeoffs exist between different integration patterns?

---

## 2. Search Stack Integration — Deep Dive

This is the **primary research focus**. The goal is to understand the best architectural patterns for coupling an LLM-based agent with an industrial-strength product search stack.

### 2.1 Query Understanding (QU)

| Component | Description | Research Focus |
|---|---|---|
| **Intent Classification** | Navigational, informational, transactional, comparison | Can LLM replace or augment rule-based intent classifiers? |
| **Entity Extraction** | Product type, brand, attributes (color, size, material) | Fine-tuned NER vs. LLM-based structured extraction |
| **Query Rewriting / Expansion** | Spelling correction, synonym expansion, query relaxation | LLM-generated query variants vs. traditional QU pipeline |
| **Constraint Parsing** | Budget ranges, compatibility requirements, exclusions | Structured output (JSON) from LLM for downstream filter application |
| **Conversational Context Fusion** | Resolving coreference across turns ("a cheaper one", "in blue") | Context window management + entity state tracking |

**Key Integration Question:** Should the agent *replace* the traditional QU pipeline (send raw NL to search) or *augment* it (LLM → structured query → existing QU)?

### 2.2 Retrieval

| Pattern | Description | Trade-offs |
|---|---|---|
| **Keyword / BM25** | Inverted index search on product titles, descriptions | High precision on exact matches; brittle on paraphrase |
| **Dense Retrieval (ANN)** | Bi-encoder embeddings (e.g., FAISS, ScaNN) over product catalog | Semantic recall; cold-start on new SKUs |
| **Hybrid Retrieval** | Sparse + dense fusion (e.g., RRF, learned interpolation) | Best of both; complexity in pipeline management |
| **Structured Filtering** | Faceted search (price, brand, ratings) applied as hard constraints | Critical for shopping; must be decoupled from ranking |
| **Multi-Index Retrieval** | Separate indexes for products, reviews, Q&A, specs | Agent selects which index to query based on user intent |

**Key Integration Patterns to Evaluate:**
- **Tool-call retrieval:** Agent emits a structured `search_products(query, filters)` tool call; search engine handles QU + retrieval; LLM handles synthesis.
- **Agentic multi-step retrieval:** Agent iteratively refines queries based on intermediate results (ReAct / Chain-of-Thought with search).
- **RAG over product catalog:** Pre-chunked product data embedded and stored; agent retrieves relevant product chunks at inference time.

### 2.3 Ranking & Re-ranking

| Stage | Traditional Approach | LLM-Augmented Approach |
|---|---|---|
| **L1 Recall** | BM25 / ANN | Same (no LLM) |
| **L2 Re-rank** | LambdaMART / LTR with behavioral features | Cross-encoder re-ranking; LLM pointwise / listwise scoring |
| **L3 Personalization** | Collaborative filtering, user embeddings | Session-aware LLM re-ranking using conversation history |
| **Diversity** | MMR, intent diversification | LLM-guided slot filling to ensure coverage of constraints |

**Research Focus:** LLM-based listwise re-ranking (PRP, RankGPT) vs. traditional LTR — latency vs. relevance tradeoffs at catalog scale.

### 2.4 Product Data Grounding & Synthesis

- **Attribute extraction at index time** vs. **LLM extraction at query time** — freshness vs. cost
- **Structured product summaries** (title, price, rating, key specs) as context chunks for the LLM
- **Hallucination prevention:** Constraining LLM generation to only reference retrieved product data (citation-backed responses)
- **Comparison tables:** LLM-synthesized structured comparisons from retrieved product attributes
- **Multi-modal grounding:** Product images as additional retrieval signals (CLIP-based visual search)

---

## 3. Agent Architecture

### 3.1 Orchestration Patterns

| Pattern | Description | Best For |
|---|---|---|
| **Single-turn RAG** | One retrieval call → LLM synthesizes response | Simple product lookup |
| **ReAct (Reason + Act)** | LLM interleaves reasoning steps with tool calls | Multi-constraint queries, comparison tasks |
| **Plan-and-Execute** | LLM generates search plan, executor runs sub-queries in parallel | Complex "gift for X" or "outfit for Y" tasks |
| **LATS / Tree Search** | LLM explores multiple search paths, backtracks on poor results | High-stakes, high-complexity shopping tasks |

### 3.2 Tool / Function Definitions
Key tools the agent should be equipped with:
- `search_products(query: str, filters: dict, top_k: int)` — primary catalog search
- `get_product_details(product_id: str)` — fetch full PDP data
- `get_reviews_summary(product_id: str, aspect: str)` — aspect-based review summarization
- `compare_products(product_ids: list, attributes: list)` — structured comparison
- `check_compatibility(product_id: str, context: dict)` — compatibility check (e.g., "fits my 2022 MacBook Pro")
- `get_deals(query: str, price_range: tuple)` — promotions / price history

### 3.3 State & Memory Management
- **In-context state:** Short-term constraint accumulation across turns (sliding window or summarization)
- **External memory:** User preference store (past purchases, saved items, style profile)
- **Catalog knowledge cache:** Pre-computed embeddings and product summaries stored in vector DB

---

## 4. Evaluation Framework

### 4.1 Retrieval Quality
- **Recall@K, NDCG@K** on shopping-specific query benchmarks
- **Constraint satisfaction rate** — % of results matching all stated filters
- **Query reformulation success** — does LLM rewriting improve retrieval over raw NL query?

### 4.2 Response Quality
- **Factual accuracy / grounding rate** — are all product claims verifiable from retrieved data?
- **Helpfulness (human eval)** — does the response help the user make a purchase decision?
- **Comparison fidelity** — accuracy of LLM-generated comparison tables vs. ground truth attributes

### 4.3 System Metrics
- **End-to-end latency** (P50/P99) — LLM inference + search latency budget
- **Cost per query** — LLM token cost vs. retrieval cost
- **Conversion rate** — downstream business metric; did the interaction lead to a purchase?

---

## 5. Key Technical Challenges

1. **Catalog scale:** Product catalogs have hundreds of millions of SKUs — LLM can't process the full catalog in-context; retrieval is non-negotiable.
2. **Freshness:** Prices, inventory, and promotions change in real-time — agent must not hallucinate stale data.
3. **Long-tail queries:** Highly specific or niche product queries where training data is sparse.
4. **Latency budget:** Users expect sub-second responses; LLM inference + multi-hop retrieval must be carefully pipelined.
5. **Safety & compliance:** Avoiding misleading product claims, price errors, or biased recommendations.
6. **Cold-start personalization:** New users have no behavioral history; agent must elicit preferences conversationally.

---

## 6. Research Milestones

| # | Milestone | Output |
|---|---|---|
| 1 | Literature review: QU + conversational search | Annotated bibliography + summary doc |
| 2 | Survey of existing shopping agent deployments | Comparison matrix of architectures |
| 3 | Prototype: Tool-call RAG agent over product catalog | Working demo with evaluation harness |
| 4 | Experiment: LLM query rewriting vs. traditional QU | Benchmark results (Recall@K, latency) |
| 5 | Experiment: LLM re-ranking vs. LTR baseline | NDCG comparison, latency cost analysis |
| 6 | Multi-turn conversation evaluation | Constraint satisfaction metrics across turns |
| 7 | System design document | Architecture diagram + design decision log |

---

## 7. Reference Papers & Resources

### Foundational
- **ANCE, DPR, ColBERT** — dense retrieval foundations
- **RankGPT, PRP** — LLM-based re-ranking
- **ReAct, Toolformer** — agentic tool use with LLMs
- **RAG (Lewis et al., 2020)** — retrieval-augmented generation

### Shopping / E-commerce Specific
- Amazon Rufus blog posts + patents
- Walmart search architecture engineering blogs
- **ESCI dataset** (Amazon) — shopping query relevance benchmark
- **TREC Product Search Track** — standardized evaluation
- **Conversational Product Search (CPS)** literature — Zou et al., Bi et al.

### System Design
- DoorDash, Instacart, Pinterest engineering blogs on embedding-based retrieval at scale
- Elasticsearch / OpenSearch hybrid retrieval patterns
- FAISS, ScaNN, Vespa — ANN library tradeoffs at catalog scale
