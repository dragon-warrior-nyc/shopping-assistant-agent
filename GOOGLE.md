# Google AI Shopping — Research Plan

## Overview

This document is a deep-dive research plan on Google's AI-powered shopping ecosystem. As of early 2026, Google has moved decisively from a keyword-based shopping engine to a **full-stack agentic commerce platform**, powered by Gemini models, the Shopping Graph, and a new open protocol for agent-to-agent commerce. The goal of this research is to reverse-engineer the architecture, understand what's public vs. inferred, and extract lessons for building a competitive shopping assistant.

---

## 1. The Big Picture: Google's Agentic Commerce Stack (as of 2026)

Google's shopping AI is not a single product — it is a **layered stack** that spans:

```
┌─────────────────────────────────────────────────────┐
│            User Surfaces                            │
│  AI Mode (Search)  │  Gemini App  │  Business Agent │
├─────────────────────────────────────────────────────┤
│            Gemini Orchestration Layer               │
│  Query understanding  │  Fan-out  │  Re-ranking     │
├─────────────────────────────────────────────────────┤
│            Shopping Graph                           │
│  50B+ listings  │  2B refreshed/hr  │  Reviews/Q&A  │
├─────────────────────────────────────────────────────┤
│            Agentic Action Layer                     │
│  UCP  │  AP2  │  A2A  │  MCP  │  Google Pay        │
└─────────────────────────────────────────────────────┘
```

**Key announced stats (NRF 2026):**
- 50 billion product listings in the Shopping Graph
- 2 billion listings refreshed **every hour**
- 90T+ monthly tokens processed on Google Cloud Vertex AI (retail customers) — 11x YoY growth from Q4 2024 → Q4 2025

---

## 2. The Shopping Graph — Product Intelligence Engine

### 2.1 What It Is
The Shopping Graph is Google's knowledge graph purpose-built for products. It is the **ground truth layer** that all Gemini-powered shopping experiences are grounded in. Without it, LLM responses would hallucinate stale or incorrect prices, inventory status, and product specs.

### 2.2 What It Contains
| Data Type | Details |
|---|---|
| **Product listings** | 50B+ SKUs from merchant feeds via Merchant Center |
| **Pricing** | Real-time prices from merchant feeds; 2B updates/hr |
| **Inventory** | In-stock status, local inventory, store availability |
| **Reviews & ratings** | Aggregated from merchant sites + Google Shopping |
| **Product attributes** | Structured specs, variants (size/color/material) |
| **Promotions** | Merchant-submitted deals, discount codes |
| **Q&A / common questions** | Newly added (2026): answers to common product questions |
| **Compatible accessories / substitutes** | New Merchant Center attributes for conversational discovery |

### 2.3 How It's Kept Fresh
- Merchant Center feeds pushed by retailers (real-time + batch)
- Crawled product data from the web
- Structured extraction from PDPs (product detail pages)
- New (2026): dozens of new **conversational commerce attributes** in Merchant Center to improve discoverability in AI Mode, Gemini, and Business Agent

### 2.4 Research Questions
- What embedding/indexing strategy does Google use to serve Shopping Graph data to Gemini at inference time?
- Is Shopping Graph served as a structured retrieval tool (function call) or pre-indexed into a vector DB that Gemini retrieves from?
- How does Google handle conflicting product data across multiple merchant feeds for the same SKU?
- What normalization/canonicalization pipeline maps diverse merchant feed formats to a unified product schema?

---

## 3. Gemini Integration — How the LLM Plugs Into Shopping

### 3.1 AI Mode in Search
AI Mode is Google's Gemini-powered search experience. For shopping queries, it replaces the traditional keyword → blue-links flow with a **conversational, visually rich response**.

**What Gemini does in AI Mode for shopping:**
- Interprets natural-language shopping queries (e.g., "cozy sweaters for happy hour in warm autumn colors")
- Decides the response *format* dynamically: shoppable image grid, comparison table, or ranked list depending on query intent
- Calls the Shopping Graph as a structured retrieval tool to ground results
- Synthesizes review insights (e.g., "how a moisturizer feels on skin") from aggregated review data
- Handles multi-turn follow-ups with coreference resolution ("a cheaper one", "in blue")

**Key architectural signal:** AI Mode is explicitly described as doing a **"fan-out"** — issuing multiple sub-searches in parallel from a single user query, then synthesizing results into one response. This is a concrete implementation of the **Plan-and-Execute** agentic pattern.

### 3.2 The Fan-Out Technique (Visual + Text)
Announced publicly (March 2026):
- Gemini acts as the **"brain"** that reasons about the query (text or image)
- Google Lens / visual search backend acts as the **"library"** of billions of web results
- For a single image or query, Gemini performs **multi-object decomposition**: identifies N objects/sub-questions
- Issues N parallel searches simultaneously (fan-out)
- Reads and synthesizes results into a single, cohesive response

**Example:** Upload a photo of a living room → Gemini identifies lamp, rug, side table, chair → fires 4 parallel Lens queries → returns shoppable results for all 4 objects at once.

**Research Questions:**
- How is fan-out count determined? Is it a fixed cap or dynamic?
- How are conflicting or low-confidence sub-results handled in the synthesis step?
- How does Gemini decide *which tools* to invoke (Lens vs. text search vs. Shopping Graph)?

### 3.3 Gemini App + Shopping Graph
As of late 2025, the Shopping Graph was brought into the **Gemini app** (gemini.google.com):
- Users can ask shopping questions in open-ended chat
- Gemini returns shoppable product listings, comparison tables, prices from across the web, and buy links
- Available to all US Gemini users

**Architectural Implication:** This is essentially **RAG over the Shopping Graph** delivered through Gemini's chat interface. The Gemini app has a Shopping Graph retrieval tool available alongside its other tools.

### 3.4 Gemini Enterprise for Customer Experience
A new B2B offering (available in preview, Jan 2026):
- Purpose-built for **agentic retail**
- Unifies search, commerce, and service touchpoints into one agentic experience
- Supported use cases: shopping assistant, support bot, agentic search, merchandising
- Partners: The Home Depot, McDonald's, Kroger (white-labeled shopping agents in retailer apps)
- Built on open standards; fully integrated with Universal Commerce Protocol (UCP)

**Key Insight:** Google is offering the *same* Gemini + Shopping Graph stack as a **white-label platform** for retailers to deploy inside their own apps — this is a direct competitor to building a custom LLM shopping agent.

---

## 4. Agentic Commerce — The Action Layer

### 4.1 Universal Commerce Protocol (UCP)
Launched January 2026 at NRF; updated March 2026.

**What it is:** An open standard for agent-to-agent commerce. Designed as a common language so any AI agent can interact with any merchant system without custom point-to-point integrations.

**Compatible with:**
- Agent2Agent (A2A)
- Agent Payments Protocol (AP2)
- Model Context Protocol (MCP)

**Current capabilities (as of March 2026):**
| Capability | Description |
|---|---|
| **Checkout** | Native buy button on Google surfaces (AI Mode, Gemini app) via Google Pay |
| **Cart** | Add/save multiple items to a cart from a single store in one agent action |
| **Catalog** | Agent retrieves real-time product details (variants, inventory, pricing) from merchant catalog |
| **Identity Linking** | Loyalty/member benefits flow through even when shopping on Google surfaces |

**Co-developed with:** Shopify, Etsy, Wayfair, Target, Walmart; endorsed by 20+ others (Adyen, Amex, Best Buy, Flipkart, Macy's, Mastercard, Stripe, The Home Depot, Visa, Zalando).

**Research Questions:**
- How does UCP handle authentication/authorization when an agent acts on behalf of a user?
- What does the UCP wire protocol look like — is it REST, gRPC, or JSON-LD over HTTP?
- How does UCP's Catalog capability interact with Google's Shopping Graph — are they synchronized or is the catalog call a real-time passthrough to the merchant?

### 4.2 Agentic Checkout
Launched November 2025 (holiday season):
- User tracks a product's price using Google Shopping price tracking
- User sets a target price + specific variant (size, color)
- When the price drops into range, Google **autonomously purchases** the item via Google Pay on the merchant's site
- User confirms purchase details once; agent executes
- Eligible merchants: Wayfair, Chewy, Quince, select Shopify merchants

**Under the hood:** This is a **delegated agent action** — Google Pay + Shopping Graph + UCP executing a purchase workflow without the user being present on the merchant's site. This is the most advanced agentic action currently live in consumer shopping.

### 4.3 "Let Google Call" (Local Inventory Agent)
Launched November 2025:
- User searches for a product "near me"
- Google's agent (powered by **Duplex + Gemini**) calls nearby stores on the user's behalf
- Agent checks: in-stock status, price, current promotions
- Results delivered via email/text summary
- Powered by Duplex's voice AI with a "big Gemini model upgrade" that:
  - Identifies the best stores to call
  - Suggests contextually relevant questions based on the product
  - Summarizes conversations into key takeaways

**Research Questions:**
- Is Duplex still primarily a voice + NLU pipeline, or has it been fully replaced by Gemini?
- How does the agent decide which stores to call (distance? inventory signals from Shopping Graph? store partnerships)?

### 4.4 Business Agent (Branded AI Agent on Search)
Launched January 2026:
- Allows brands to run a **branded conversational agent** directly on Google Search
- Appears as a chat interface on search results for brand-related queries
- Acts as a "virtual sales associate" — answers product questions in the brand's voice
- Retailers configure and activate via Merchant Center
- Roadmap: brands will be able to train the agent on their own data, access customer insights, enable direct agentic checkout from the agent

**Participating at launch:** Lowe's, Michaels, Poshmark, Reebok.

---

## 5. Visual Search & Multimodal Product Discovery

### 5.1 Google Lens
Google Lens is the visual search engine that powers product discovery from images. Key capabilities:
- **Object recognition** for product identification
- **Style matching** — find visually similar products across the web
- **Text-in-image extraction** — scan barcodes, labels, packaging

### 5.2 Circle to Search (February 2026 update)
- Previously: circle one object in an image → one search result
- Now: circle an entire outfit/scene → AI decomposes into multiple objects → simultaneous fan-out searches → results for all components at once

**Research Questions:**
- What object detection model underlies the decomposition step — is it a dedicated vision model or Gemini's native multimodal reasoning?
- How are visual embeddings for product matching stored and served at scale?
- What is the CLIP-like model used for visual similarity search in Google Lens?

### 5.3 "Virtual Try-On"
- Uses generative AI to show how clothing looks on diverse body types
- Integrates with Shopping Graph to pull real product images from merchant feeds
- Essentially an image-conditioned diffusion model grounded in real product inventory

---

## 6. Advertising Integration — The Revenue Layer

Understanding the ad/organic boundary is critical for understanding why Google builds what it builds.

### 6.1 Direct Offers (New, Jan 2026 pilot)
- Advertisers submit exclusive offers (e.g., "20% off") to be shown in AI Mode
- Google AI determines *when* to show the offer based on query relevance — not just keyword match
- Labeled "Sponsored deal" in AI Mode results
- Expanding beyond discounts to bundles, free shipping, etc.
- Pilot partners: Petco, e.l.f. Cosmetics, Samsonite, Rugs USA, Shopify merchants

### 6.2 Commerce Media Suite
- Retailer first-party data + YouTube advertising unified
- Enables closed-loop measurement: from discovery ad → purchase
- Relevant for understanding how Google monetizes the entire shopping funnel

### 6.3 Research Questions
- How does Google prevent the ad layer from degrading trust in AI Mode responses?
- Is there a ranking firewall between paid (Direct Offers) and organic (Shopping Graph) results in AI Mode?
- How does the agent decide when to surface a sponsored result vs. an organic one in a conversational context?

---

## 7. Infrastructure & Technical Architecture (Inferred)

Much of this is not publicly disclosed, but can be inferred from patents, research papers, and engineering blog posts.

### 7.1 Shopping Graph as a Retrieval Tool
**Hypothesis:** Gemini does not directly embed the 50B product catalog into its context window. Instead, it has access to a **structured query API** over the Shopping Graph (a function/tool call), which returns a ranked list of product objects matching a structured query (query text + filters). This is equivalent to the `search_products(query, filters, top_k)` tool pattern.

**Evidence:** Google's own descriptions consistently describe Shopping Graph as the *source* that AI Mode responses are "powered by" — implying tool-call retrieval, not in-context injection.

### 7.2 Freshness Architecture
- Product data refreshed at 2B listings/hour implies a **streaming ingestion pipeline** (likely Pub/Sub + Dataflow or equivalent)
- Gemini retrieval must be hitting a **low-latency serving layer** (not batch-indexed), likely backed by something like Spanner or Bigtable for structured lookups + a dense index (ScaNN) for semantic search

### 7.3 Gemini Model Routing
For shopping queries, Google likely routes to different Gemini configurations:
- **Simple product lookup** → lightweight Gemini Flash model + Shopping Graph tool call
- **Comparison / complex reasoning** → Gemini Pro/Ultra + multi-step fan-out
- **Visual search** → Gemini multimodal model + Lens backend
- **Agentic checkout** → Gemini + UCP tool calls + AP2 + Google Pay

### 7.4 Grounding & Hallucination Prevention
- All product facts (price, availability, specs) must be sourced from Shopping Graph, not generated
- Likely implemented as **citation enforcement**: Gemini can only state product claims that are attributed to a retrieved product record
- Price/inventory data is the most freshness-critical — likely served from a real-time cache, not the main embedding index

---

## 8. Key Research Questions (Prioritized)

### Tier 1 — Architecture (Most Valuable)
1. What is the exact interface between Gemini and the Shopping Graph? (structured API vs. embedding retrieval vs. hybrid?)
2. How is the fan-out query planning implemented? (explicit planning step? Chain-of-Thought? Hardcoded?)
3. How does Google prevent LLM hallucination of product facts (price, availability) in grounded responses?
4. How is conversational context (multi-turn constraints) accumulated and passed to Shopping Graph queries?

### Tier 2 — Product Intelligence
5. What new "conversational commerce attributes" are being added to Merchant Center and how do they improve retrieval in AI Mode?
6. How does Google canonicalize product entities across millions of different merchant feeds (same product, different titles/descriptions)?
7. How does aspect-based review summarization work at Shopping Graph scale?

### Tier 3 — Agentic Infrastructure
8. What does the UCP wire protocol specification look like at a technical level?
9. How does Google handle agent authorization and user consent for delegated actions (agentic checkout, "Let Google Call")?
10. How does the Business Agent's retrieval work — does it use the retailer's own catalog data or the Shopping Graph, or both?

---

## 9. Research Milestones

| # | Milestone | Output |
|---|---|---|
| 1 | Deep read of all public Google Shopping/AI blog posts and NRF 2026 announcements | Annotated timeline + feature inventory |
| 2 | Review Google Research papers on Shopping Graph, product entity resolution, and review summarization | Technical notes doc |
| 3 | Study UCP spec at [ucp.dev](https://ucp.dev) — wire protocol, capabilities, authentication model | UCP protocol summary |
| 4 | Review Google patents on Shopping Graph indexing, query understanding for shopping, and Duplex | Patent analysis |
| 5 | Analyze AI Mode's fan-out behavior empirically — test multi-object queries, observe response structure | Behavioral analysis + screenshots |
| 6 | Review Gemini API shopping-related function calling patterns and tool definitions | Tool schema analysis |
| 7 | Compare Google's approach vs. Amazon Rufus and Walmart Sparky on key dimensions | Comparative architecture matrix |
| 8 | Synthesize into design recommendations for own shopping assistant | Design decisions doc |

---

## 10. Key Sources & References

### Google Official
- [NRF 2026 Sundar Pichai remarks](https://blog.google/company-news/inside-google/message-ceo/nrf-2026-remarks/) — Shopping Graph stats, UCP launch, Gemini Enterprise for CX
- [Agentic commerce tools for retailers (Jan 2026)](https://blog.google/products/ads-commerce/agentic-commerce-ai-tools-protocol-retailers-platforms/) — UCP details, Business Agent, Direct Offers, Merchant Center attributes
- [Holiday shopping AI update (Nov 2025)](https://blog.google/products/shopping/agentic-checkout-holiday-ai-shopping/) — AI Mode shopping, Gemini app shopping, agentic checkout, "Let Google Call"
- [UCP March 2026 updates](https://blog.google/products-and-platforms/products/shopping/ucp-updates/) — Cart, Catalog, Identity Linking capabilities
- [Circle to Search fan-out (Mar 2026)](https://blog.google/company-news/inside-google/googlers/how-google-ai-visual-search-works/) — Fan-out technique, Gemini multimodal visual search
- [UCP spec](https://ucp.dev) — Open protocol specification
- [Google Merchant Center](https://merchants.google.com) — Feed attributes, conversational commerce attributes

### Research Papers (Google)
- **Shopping Queries Dataset (ESCI)** — product search relevance at scale
- **PaLM-based product attribute extraction** — Google Research
- **Duplex** — voice AI agent for phone calls (underpins "Let Google Call")
- **ScaNN** — Google's ANN library for dense retrieval at scale
- **MUM (Multitask Unified Model)** — predecessor multimodal understanding model
- **Gemini technical report** — multimodal capabilities relevant to visual shopping

### Third-Party Analysis
- Stripe's NRF 2026 blog on agentic commerce trends
- Salesforce announcement on UCP integration
- Commerce Inc. UCP press release
