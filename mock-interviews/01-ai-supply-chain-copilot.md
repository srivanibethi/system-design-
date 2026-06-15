# Mock Interview 1: AI-Powered Supply Chain Copilot

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "We want to build an AI assistant for supply chain managers. They should be able to ask natural language questions about their operations — things like 'Why did we stockout last week?', 'What's our supplier risk exposure?', or 'Recommend optimal reorder quantities for next month.' Design this system end-to-end."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Users & Scale:**
1. "Who are the primary users — supply chain analysts, VPs, warehouse managers? Their technical sophistication affects UX complexity."
2. "How many enterprise customers do we support? Are we multi-tenant?"
3. "What's the expected concurrent user count and query volume?"
4. "Is this a greenfield product or are we augmenting an existing tool?"

**About Data Sources:**
5. "What systems hold the data — SAP, Oracle, custom ERPs? Do we have direct access or go through APIs?"
6. "Is the data structured (SQL tables), unstructured (documents), or both?"
7. "How fresh does the data need to be — real-time or is daily refresh acceptable?"
8. "How much historical data are we talking about? Years? Terabytes?"

**About AI Behavior:**
9. "What's the accuracy bar? Enterprise users making financial decisions need high confidence."
10. "Should the system ever take actions autonomously, or always recommend?"
11. "Do we need to handle multi-turn conversations (follow-up questions)?"
12. "What languages do we need to support?"

**About Constraints:**
13. "Are there compliance requirements — SOC 2, GDPR, data residency?"
14. "What's the latency target for responses — is 10-15 seconds acceptable for complex queries?"
15. "Budget constraints on LLM API costs?"

### Interviewer's Likely Answers:
- Google-scale platform: 10,000+ enterprise customers globally (think: Google Supply Chain AI as a service)
- 500K-1M concurrent users across all tenants
- Mix of structured (ERP data, 100B+ rows) and unstructured (10M+ documents per large customer)
- Real-time streaming for all data, sub-minute freshness
- Must cite sources, high accuracy, recommend AND auto-act for approved workflows
- SOC 2, ISO 27001, FedRAMP, GDPR, data residency per region (US, EU, APAC)
- <3 seconds for simple lookups, <8 seconds for complex analytical queries
- Multi-region deployment (us-central1, europe-west4, asia-east1)

---

## PHASE 2: Requirements Summary & Back-of-Envelope (3 minutes)

### State This Explicitly:

```
"Let me summarize what I'm hearing and do some quick math:

FUNCTIONAL REQUIREMENTS:
1. Natural language Q&A grounded in customer's supply chain data
2. Three query types: factual (lookup), analytical (aggregation/trends), 
   recommendation (optimization-backed)
3. Multi-turn conversation with context retention
4. Cited sources for every answer
5. Audit trail for compliance

NON-FUNCTIONAL:
- 10K customers × 100 active users × ~8 queries/hour = 8M queries/hour = ~2,200 QPS avg
- Peak (Monday mornings, month-end, global overlap hours): 10x = 22,000 QPS
- Burst: Black Friday/peak planning season: 50,000 QPS
- Latency: p50 <2s, p95 <5s, p99 <10s (even for complex queries)
- Availability: 99.99% (4.3 min downtime/month) — Google-grade SLO
- Data isolation: Tenant A must NEVER see Tenant B's data (crypto-sharded)
- Multi-region: active-active across 3+ regions for data residency

COST ESTIMATE:
- 8M queries/hour × $0.008/query (with aggressive caching + model routing) = $64K/hour = $1.5M/day
- With 80% semantic cache hit + model routing (70% to smaller models): $300K/day
- At Google scale, amortized cost per customer: $30K/year → high-margin at enterprise pricing ($200K+/year)
- Must justify: $110M/year run cost for $2B+ ARR business

Does this scope feel right to you?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Draw This Diagram:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │
│  │ Web App (React)│  │ Chat Widget  │  │ API (for programmatic access)│  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │
└─────────┼──────────────────┼─────────────────────────┼──────────────────┘
          │                  │                         │
          ▼                  ▼                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                                        │
│  Auth (JWT) │ Rate Limiting │ Tenant Routing │ Request Logging           │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION SERVICE                                  │
│                                                                          │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────────────┐ │
│  │ Session Mgr  │    │ Query Classifier  │    │ Response Synthesizer  │ │
│  │ (context,    │    │ (route to right   │    │ (format, cite,       │ │
│  │  history)    │    │  pipeline)        │    │  confidence score)   │ │
│  └──────────────┘    └────────┬─────────┘    └───────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐
│ RAG PIPELINE │ │ ANALYTICS    │ │ RECOMMENDATION       │
│              │ │ PIPELINE     │ │ PIPELINE             │
│ Vector DB +  │ │ Text-to-SQL  │ │ Optimization Engine  │
│ Doc Store    │ │ + Execution  │ │ + ML Models          │
│ + Reranking  │ │ + Viz        │ │ + Constraints Solver │
└──────────────┘ └──────────────┘ └──────────────────────┘
        │               │                    │
        └───────────────┼────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       DATA LAYER (Per Tenant)                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐   │
│  │ Vector DB  │  │ Analytics  │  │ Feature    │  │ Document       │   │
│  │ (embeddings│  │ Warehouse  │  │ Store      │  │ Store (S3)     │   │
│  │  per tenant│  │ (BigQuery) │  │ (Redis)    │  │                │   │
│  └────────────┘  └────────────┘  └────────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
        │                                     ▲
        │                                     │
┌───────┴─────────────────────────────────────┼───────────────────────────┐
│                   INGESTION LAYER            │                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  ┌───────────────┐       │
│  │ ERP CDC  │  │ Doc      │  │ API      │  │  │ Data Quality  │       │
│  │ Connector│  │ Processor│  │ Poller   │  │  │ Pipeline      │       │
│  └──────────┘  └──────────┘  └──────────┘  │  └───────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Explain the Flow:

```
"Here's the happy path for a query like 'Why did we stockout on SKU-789?':

1. User types question in chat UI
2. API Gateway authenticates, identifies tenant, rate-checks
3. Orchestration Service:
   a. Session Manager loads conversation history
   b. Query Classifier determines: this is ANALYTICAL (needs data lookup + reasoning)
   c. Routes to BOTH RAG pipeline (for context/documents) AND Analytics pipeline (for data)
4. Analytics Pipeline:
   - Translates to SQL: SELECT inventory_levels, demand, supply_orders WHERE sku='789' AND date=last_week
   - Executes against tenant's warehouse
   - Returns structured data
5. RAG Pipeline:
   - Retrieves relevant documents (reorder policies, supplier SLAs)
   - Provides contextual background
6. Response Synthesizer:
   - Combines data + context + conversation history
   - Generates explanation with citations
   - Assigns confidence score
7. Returns to user with sources cited

The key insight: we don't treat every query the same. The classifier 
routes to specialized pipelines, and the synthesizer combines their outputs."
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Query Classification & Routing

**Interviewer might ask**: "How does the classifier work? What if it misroutes?"

```
"The Query Classifier is a critical component. Here's how I'd design it:

APPROACH: Fine-tuned small model (not LLM — too slow/expensive for routing)

┌─────────────────────────────────────────────────┐
│           QUERY CLASSIFICATION                   │
├─────────────────────────────────────────────────┤
│ Input: "Why did we stockout on SKU-789?"         │
│                                                  │
│ Step 1: Intent Detection (fine-tuned BERT)       │
│   → intent: ROOT_CAUSE_ANALYSIS                  │
│   → confidence: 0.94                             │
│                                                  │
│ Step 2: Entity Extraction                        │
│   → entities: {sku: "SKU-789", event: "stockout",│
│                time: "last_week"}                 │
│                                                  │
│ Step 3: Pipeline Routing                         │
│   → primary: ANALYTICS (needs data)              │
│   → secondary: RAG (needs context/policies)      │
│   → exclude: RECOMMENDATION (not asking for advice)│
│                                                  │
│ Step 4: Confidence Check                         │
│   → if confidence < 0.7: ask clarifying question │
│   → if ambiguous: route to multiple pipelines    │
└─────────────────────────────────────────────────┘

CATEGORIES:
- FACTUAL: "What's the lead time for supplier X?" → RAG
- ANALYTICAL: "What were our top 10 stockouts last month?" → Text-to-SQL
- DIAGNOSTIC: "Why did X happen?" → Analytics + RAG (combined)
- PREDICTIVE: "Will we stockout next week?" → ML Model Serving
- RECOMMENDATION: "What should I reorder?" → Optimization Engine
- CONVERSATIONAL: "Can you explain that further?" → Context from session

MISROUTING MITIGATION:
1. Low-confidence queries → fan out to multiple pipelines, synthesize
2. Feedback loop: user 👎 triggers reclassification analysis
3. Fallback: if specialized pipeline fails, fall back to general RAG
4. A/B test routing decisions, measure satisfaction per route

The classifier adds <50ms latency (it's a small model). The cost of 
misrouting is higher latency (ran wrong pipeline first), not incorrect 
answers (synthesizer validates before responding)."
```

---

### Deep Dive 2: Multi-Tenant Data Isolation

**Interviewer might ask**: "How do you ensure tenant A never sees tenant B's data?"

```
"This is the #1 security requirement. I'd use defense-in-depth:

LAYER 1: Authentication & Context
┌────────────────────────────────────┐
│ Every request carries tenant_id     │
│ Extracted from JWT at API gateway   │
│ Propagated via request context      │
│ (like a thread-local / context var) │
└────────────────────────────────────┘

LAYER 2: Data Isolation Model
┌────────────────────────────────────────────────────────┐
│ OPTION A: Shared DB, tenant_id column                   │
│   Pro: Simpler ops, cheaper                             │
│   Con: Risk of query without WHERE tenant_id=X          │
│                                                         │
│ OPTION B: Separate schemas per tenant                   │
│   Pro: Stronger isolation, easy to reason about         │
│   Con: More operational complexity                      │
│                                                         │
│ OPTION C: Separate databases per tenant                 │
│   Pro: Strongest isolation, independent scaling         │
│   Con: Expensive, hard to manage at 100+ tenants        │
│                                                         │
│ MY CHOICE: Option B (separate schemas) with             │
│ per-tenant encryption keys for data at rest.            │
│ Reason: Strong isolation without ops explosion.         │
│ We can shard large tenants to their own DB later.       │
└────────────────────────────────────────────────────────┘

LAYER 3: Vector DB Isolation
- Namespace per tenant in vector DB (e.g., Pinecone namespaces)
- Every vector search MUST include tenant namespace filter
- Embedding models are shared (stateless), but stored embeddings are isolated

LAYER 4: LLM Prompt Isolation
- Tenant data is injected into context window per-request
- No fine-tuning on multi-tenant data (avoids memorization leakage)
- Prompt includes: "Only answer based on the provided context. 
  Do not reference any information not in the context."

LAYER 5: Automated Verification
- Integration tests that attempt cross-tenant access (must fail)
- Audit logging: every data access logged with tenant context
- Quarterly penetration testing

The paranoia level here is appropriate because a data leak between 
enterprise customers is an existential business risk."
```

---

### Deep Dive 3: RAG Pipeline Quality

**Interviewer might ask**: "How do you ensure high-quality answers? What about hallucination?"

```
"RAG quality is a pipeline problem with multiple stages to optimize:

┌─────────────────────────────────────────────────────────────┐
│               RAG QUALITY PIPELINE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAGE 1: RETRIEVAL QUALITY                                 │
│  ─────────────────────────                                  │
│  Problem: Retrieve the RIGHT documents                       │
│                                                              │
│  Query: "Why did we stockout on SKU-789?"                   │
│         ↓                                                    │
│  Query Expansion: "stockout SKU-789 inventory depletion      │
│                    out-of-stock shortage"                     │
│         ↓                                                    │
│  Hybrid Search:                                              │
│    Dense (semantic): Embed query → vector search (top 20)    │
│    Sparse (keyword): BM25 search for "SKU-789" (top 20)     │
│    Metadata filter: tenant_id=X, doc_type IN (inventory,     │
│                     reorder_policy, supplier_info)            │
│         ↓                                                    │
│  Reciprocal Rank Fusion → Merged top 20                      │
│         ↓                                                    │
│  Cross-Encoder Reranking → Top 5                             │
│                                                              │
│  METRICS: Recall@5, MRR, NDCG                               │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAGE 2: GENERATION QUALITY                                 │
│  ───────────────────────────                                 │
│  Problem: Generate FAITHFUL answer from retrieved context     │
│                                                              │
│  Prompt Structure:                                           │
│  ┌──────────────────────────────────────────────────┐       │
│  │ SYSTEM: You are a supply chain analyst assistant.  │       │
│  │ Answer ONLY from the provided context.             │       │
│  │ If the context doesn't contain the answer,         │       │
│  │ say "I don't have enough information."             │       │
│  │ Always cite which source supports each claim.      │       │
│  │                                                    │       │
│  │ CONTEXT: [5 retrieved chunks with source IDs]      │       │
│  │                                                    │       │
│  │ CONVERSATION HISTORY: [prior turns]                │       │
│  │                                                    │       │
│  │ USER: Why did we stockout on SKU-789?              │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  Output Schema (enforced via function calling):              │
│  {                                                           │
│    "answer": "...",                                          │
│    "citations": [{"source_id": "doc_123", "quote": "..."}], │
│    "confidence": 0.85,                                       │
│    "reasoning": "Based on inventory data showing..."         │
│  }                                                           │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAGE 3: VERIFICATION                                       │
│  ─────────────────────                                       │
│  Problem: Catch hallucinations before reaching user          │
│                                                              │
│  Check 1: Citation verification                              │
│    - For each citation, verify the quote exists in source    │
│    - If any citation is fabricated → regenerate or warn      │
│                                                              │
│  Check 2: Confidence threshold                               │
│    - confidence < 0.6 → "I'm not confident about this.      │
│      Here's what I found, but please verify: ..."            │
│    - confidence < 0.3 → "I don't have enough information     │
│      to answer this reliably."                               │
│                                                              │
│  Check 3: Consistency check (for critical queries)           │
│    - Generate answer twice with different retrieved docs      │
│    - If answers contradict → flag for human review           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

EVALUATION FRAMEWORK:
┌──────────────────────────────────────────────────┐
│ OFFLINE (weekly):                                 │
│ - 200 golden Q&A pairs per customer segment       │
│ - Measure: faithfulness, relevancy, completeness  │
│ - Automated via LLM-as-judge (RAGAS framework)    │
│                                                   │
│ ONLINE (continuous):                              │
│ - User feedback: 👍/👎 on every answer            │
│ - Click-through on citations (did user verify?)   │
│ - Session length (longer = engaged OR confused)   │
│ - Escalation rate (asked a human after AI answer) │
│                                                   │
│ ALERTING:                                         │
│ - Satisfaction drops below 80% → page oncall      │
│ - Hallucination detected → immediate investigation│
│ - New topic cluster with low confidence → gap in  │
│   knowledge base                                  │
└──────────────────────────────────────────────────┘
```

---

## PHASE 5: Operational Concerns & Wrap-Up (5 minutes)

### Monitoring & Observability
```
"For production readiness, I'd instrument:

KEY METRICS:
- Query latency percentiles: p50, p95, p99 (target: <5s/<15s/<30s)
- Answer quality: daily avg confidence score, 👍 rate
- LLM cost per query: track by query type and customer tier
- Retrieval metrics: avg docs retrieved, reranking changes, cache hit rate
- Error rate: LLM timeouts, retrieval failures, empty results

TRACING (OpenTelemetry):
User query → classify (50ms) → retrieve (200ms) → rerank (100ms) → 
  generate (3-8s) → verify (500ms) → respond

Every trace includes: tenant_id, query_type, model_used, tokens_consumed,
                      docs_retrieved, confidence_score, user_feedback

ALERTING:
- p99 latency > 30s for 5 minutes → page oncall
- Confidence score avg drops 10% day-over-day → investigate
- LLM provider error rate > 5% → activate fallback provider
"
```

### Deployment Strategy
```
"I'd deploy incrementally:

PHASE 1 (Month 1): 3 design partners, RAG-only (factual Q&A)
  - Learn: What questions do they actually ask?
  - Validate: Is retrieval quality sufficient?

PHASE 2 (Month 2-3): Add analytics pipeline (Text-to-SQL)
  - Learn: How complex are their analytical needs?
  - Validate: SQL accuracy + safety

PHASE 3 (Month 4-6): Add recommendations, scale to 20 customers
  - Learn: Do they trust AI recommendations?
  - Validate: Recommendation quality vs current process

Canary deployment: New versions get 5% traffic, monitor for quality 
regression, promote to 100% after 24h of green metrics."
```

### Extensions (What V2 Looks Like)
```
"If we had more time, I'd explore:

1. PROACTIVE INSIGHTS: System detects anomalies and pushes alerts
   ('I noticed demand for SKU-789 spiked 3x today — want me to 
   investigate?')

2. COLLABORATIVE FEATURES: Share an AI finding with a colleague,
   they can ask follow-up questions with full context

3. LEARNING FROM ACTIONS: Track which recommendations users act on,
   use this as reward signal for improving recommendations

4. MULTI-MODAL: Accept photos (damaged goods, warehouse layouts),
   generate charts/visualizations in responses

5. AUTONOMOUS AGENTS: For low-risk decisions below $X threshold,
   allow the system to auto-execute (e.g., reorder standard supplies)"
```

---

## Scoring Rubric (Self-Evaluate)

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Scoping | Quantified scale, identified 3 query types, stated explicit assumptions | ☐ |
| Architecture | Clean separation of concerns, right components for the problem | ☐ |
| AI Depth | RAG pipeline details, hallucination mitigation, evaluation framework | ☐ |
| Multi-tenancy | Defense-in-depth isolation, not just "add tenant_id column" | ☐ |
| Trade-offs | Quantified cost, compared approaches, stated reasoning | ☐ |
| Operations | Monitoring, deployment strategy, failure modes | ☐ |
| Staff signals | Drove conversation, proposed phased rollout, org implications | ☐ |
| Communication | Structured, clear, invited discussion | ☐ |
# ENHANCED: AI Supply Chain Copilot — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 1 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES (GOOGLE-SCALE)

### Query Volume Modeling
```
USERS:
- 10,000 enterprise customers (Fortune 5000 + large mid-market)
- Avg 100 active users per customer = 1,000,000 total registered users
- ~300K DAU (30% daily active rate for enterprise tools)
- Power users: 20% send 80% of queries
- Avg session: 5 queries in 15 minutes, 6 sessions/day (stickier than chat)

QUERY MATH:
- Global platform: 24/7 usage across timezones (no quiet period)
- Average: 300K DAU × 30 queries/day = 9M queries/day = ~105 QPS steady
- Business hours amplification: 3x during global overlap (EU+US: 1-5pm UTC)
- Peak (Monday mornings, month-end close, quarter-end): 10x = 1,050 QPS
- Burst (Black Friday, Chinese New Year, major disruptions): 50x = 5,250 QPS
- Absolute max design target: 10,000 QPS (headroom for growth)

THIS IS HIGH QPS for LLM systems. The challenges are:
1. LLM serving at 10K QPS: need massive GPU fleet or self-hosted models
2. Cost per query at scale: $0.025/query × 9M/day = $225K/day UNOPTIMIZED
3. Vector search at 10K QPS across 200B+ vectors
4. Multi-tenancy isolation at speed (can't add per-query auth overhead)
5. Global latency: need regional deployments, not single-region

CRITICAL DESIGN DECISION:
  At Google scale, you CANNOT rely solely on external LLM APIs (OpenAI, Anthropic).
  You need:
  - Self-hosted models (Gemini/PaLM on TPUs) for 80% of traffic (simple queries)
  - Frontier model APIs (GPT-4/Claude) only for complex reasoning (20%)
  - Aggressive semantic caching (80%+ hit rate for repeated patterns)
  - Model cascade: smallest model that can handle each query type
```

### Storage Modeling
```
PER CUSTOMER (Large Enterprise — top 10%):
- ERP data (structured): 5B rows × 200 bytes = 1 TB
- Documents (unstructured): 500K docs × 300KB = 150 GB
- Embeddings: 500K docs × 50 chunks × 6KB = 150 GB
- Conversation history: 100K sessions × 10 turns × 3KB = 3 GB
- Customer knowledge graph: 50M nodes × 500B = 25 GB
- Total per large customer: ~1.3 TB

PER CUSTOMER (Typical — median):
- ERP data: 500M rows × 200 bytes = 100 GB
- Documents: 50K docs × 250KB = 12.5 GB
- Embeddings: 50K docs × 30 chunks × 6KB = 9 GB
- Conversation history: 10K sessions × 8 turns × 3KB = 240 MB
- Total per median customer: ~125 GB

ALL CUSTOMERS (PLATFORM TOTAL):
- Top 500 customers × 1.3 TB = 650 TB
- Next 2,000 × 300 GB = 600 TB
- Remaining 7,500 × 125 GB = 940 TB
- TOTAL MANAGED DATA: ~2.2 PB
- Growth: 40%/year (new customers + data volume growth)
- 3-year projection: ~6 PB

VECTOR DB SIZING:
- Total chunks: 10K customers × avg 1M chunks = 10B vectors
- Storage: 10B × 1536-dim × float16 = 30 TB raw
- With HNSW index overhead (2.5x): 75 TB
- CANNOT fit in memory — need distributed vector DB:
  Option A: Google Vertex AI Matching Engine (managed, scales to billions)
  Option B: Weaviate/Milvus on GKE (50-node cluster, 1.5TB RAM each)
  Option C: Custom ScaNN-based solution (Google's internal vector search)
- MY CHOICE: Vertex AI Matching Engine for ease + Google Cloud Credits,
  with per-tenant index sharding for isolation

MULTI-REGION REPLICATION:
- 3 active regions: US, EU, APAC
- Full data replicated per region for data residency (6.6 PB total w/ 3x)
- Vector indices: regional (queries go to nearest region)
- Structured data: Spanner (global consistency, regional reads)
```

### Cost Modeling
```
LLM COSTS AT GOOGLE SCALE:
┌──────────────────────────────────────────────────────────────────────────┐
│ Component              │ Volume      │ Unit Cost │ Daily Cost │ Strategy │
├──────────────────────────────────────────────────────────────────────────┤
│ Self-hosted Gemini     │ 5.4M q/day  │ $0.003    │ $16K       │ 60% of  │
│ (TPU pods, simple)     │ (60%)       │           │            │ queries  │
├──────────────────────────────────────────────────────────────────────────┤
│ Self-hosted Gemini Pro │ 1.8M q/day  │ $0.008    │ $14K       │ 20%     │
│ (TPU pods, medium)     │ (20%)       │           │            │ complex  │
├──────────────────────────────────────────────────────────────────────────┤
│ Frontier API (Claude/  │ 900K q/day  │ $0.025    │ $22K       │ 10%     │
│ GPT-4, hardest queries)│ (10%)       │           │            │ hardest  │
├──────────────────────────────────────────────────────────────────────────┤
│ Semantic cache hit     │ 900K q/day  │ $0.0005   │ $450       │ 10%     │
│ (no LLM call needed)   │ (10%)       │           │            │ repeated │
├──────────────────────────────────────────────────────────────────────────┤
│ TOTAL LLM              │ 9M q/day    │ $0.006avg │ $52K/day   │          │
│ MONTHLY LLM            │ 270M q/mo   │           │ $1.6M/mo   │          │
└──────────────────────────────────────────────────────────────────────────┘

INFRASTRUCTURE COSTS:
┌────────────────────────────────────────────────────────────────────────┐
│ Component                   │ Spec                    │ Monthly Cost   │
├─────────────────────────────┼─────────────────────────┼────────────────┤
│ TPU pods (self-hosted LLMs) │ 16× v5e pods (256 chips)│ $800K          │
│ Vector DB (Vertex Matching) │ 75 TB, 3 regions        │ $200K          │
│ Spanner (structured data)   │ 2 PB, multi-region      │ $400K          │
│ GCS (document storage)      │ 3 PB (w/ replication)   │ $60K           │
│ Redis (caching layer)       │ 500 nodes, 50TB total   │ $150K          │
│ GKE (app/orchestration)     │ 200 nodes, 3 regions    │ $100K          │
│ Networking (cross-region)   │ ~50 TB/month egress     │ $100K          │
│ Monitoring/observability    │ Cloud Ops at scale      │ $50K           │
├─────────────────────────────┼─────────────────────────┼────────────────┤
│ TOTAL INFRASTRUCTURE        │                         │ ~$1.86M/month  │
│ + LLM COSTS                 │                         │ ~$1.6M/month   │
│ GRAND TOTAL                 │                         │ ~$3.5M/month   │
└────────────────────────────────────────────────────────────────────────┘

UNIT ECONOMICS:
- Cost per customer: $3.5M ÷ 10K = $350/month avg
- Revenue per customer: $15K-200K/year (tiered pricing)
- Blended margin: 70-85% at scale (Google infrastructure advantage)
- Break-even at: ~2,000 customers

SCALING ECONOMICS (why this works at Google):
- Self-hosted LLMs on TPUs are 5-10x cheaper than API calls at scale
- Spanner/Bigtable pricing is internal (much cheaper than external cloud)
- Shared infrastructure amortized across Google Cloud customers
- Semantic caching improves with scale (more users → more cache hits)
```

---

## ALTERNATIVE ARCHITECTURES (Pick One, Defend It)

### Approach A: Monolithic LLM-First (Simple)
```
┌──────────────────────────────────────────┐
│ User Query → Single LLM Call → Response  │
│                                          │
│ LLM has access to tools:                 │
│ - sql_query(query) → execute SQL         │
│ - search_docs(query) → vector search     │
│ - get_metrics(metric) → real-time data   │
│                                          │
│ LLM decides: which tools to call,        │
│ in what order, how to synthesize          │
└──────────────────────────────────────────┘

PROS:
+ Simplest architecture (one service, one LLM call with tool use)
+ LLM handles routing implicitly (no classifier needed)
+ Easy to add new tools/capabilities
+ Fastest time to market (2-3 months)
+ Naturally handles multi-step reasoning

CONS:
- LLM tool selection is unreliable at scale (wrong tool 10-15% of time)
- Entire latency depends on LLM's multi-step reasoning (15-30 seconds)
- Hard to optimize individual pipelines independently
- Cost: every query uses expensive model (no routing to cheaper models)
- Debugging is opaque (why did it choose that tool?)
- Hard to set per-pipeline SLAs

WHEN TO CHOOSE: Early-stage startup, <10 customers, proving product-market fit.
```

### Approach B: Pipeline Router (What I Recommended in Main File)
```
┌──────────────────────────────────────────────────────────┐
│ User Query → Classifier → Specialized Pipeline → LLM Synthesis │
│                                                          │
│ Pipelines:                                               │
│ - RAG pipeline (document Q&A)                            │
│ - Analytics pipeline (Text-to-SQL)                       │
│ - Recommendation pipeline (optimization)                 │
│ - Conversational pipeline (follow-ups)                   │
│                                                          │
│ Each pipeline independently optimized                    │
└──────────────────────────────────────────────────────────┘

PROS:
+ Each pipeline optimized independently (different latency, accuracy, cost)
+ Classifier is cheap and fast (<50ms)
+ Can use different models per pipeline (cheap for simple, expensive for complex)
+ Clear debugging: know exactly which pipeline ran
+ Independent scaling (analytics heavy? scale that pipeline)
+ Team can own pipelines independently

CONS:
- More architectural complexity (multiple services to maintain)
- Classifier errors cascade (wrong pipeline → bad answer)
- Multi-pipeline queries need orchestration (query needs BOTH data + docs)
- 3-4 months to build properly
- May miss emergent capabilities that monolithic approach discovers

WHEN TO CHOOSE: Growth stage, 20+ customers, need reliability and cost optimization.
```

### Approach C: Agent-Based (Most Capable)
```
┌──────────────────────────────────────────────────────────────┐
│ User Query → Planner Agent → [Sub-Agents] → Synthesizer     │
│                                                              │
│ Planner decomposes query into sub-tasks:                     │
│ "Why stockout?" → 1. Get inventory history                   │
│                   2. Get demand vs forecast                   │
│                   3. Get supplier delivery status             │
│                   4. Check if safety stock was adequate       │
│                                                              │
│ Each sub-task → specialized agent → results                  │
│ Synthesizer combines sub-results → final answer              │
└──────────────────────────────────────────────────────────────┘

PROS:
+ Handles complex multi-step queries naturally
+ Can reason about what information is needed
+ Recovers from partial failures (one sub-agent fails, others still contribute)
+ Most accurate for diagnostic/root-cause queries
+ Scales capability without architecture changes

CONS:
- Slowest: multiple sequential LLM calls (20-60 seconds)
- Most expensive: 3-5x more tokens per query
- Hardest to debug (agent chains are opaque)
- Non-deterministic (same query can take different paths)
- Reliability: chain of LLM calls → compound error rate
- 6+ months to build robustly

WHEN TO CHOOSE: Mature product, complex use cases, accuracy > latency.
```

### My Recommendation & Why (At Google Scale):
```
CHOOSE Approach B (Pipeline Router) as the backbone because:
1. At 10K QPS, you NEED independent pipeline scaling (analytics traffic 
   bursts independently from RAG traffic)
2. Cost control is existential at $3.5M/month — model routing saves 60%+
3. At 10K customers, pipeline failures must be isolated (one bad pipeline 
   can't take down the platform)
4. 50+ engineer team can own pipelines independently (org-scalable)
5. Each pipeline gets its own SLO, capacity planning, and on-call rotation

LAYER Approach C (Agent) for complex queries (10-15% of traffic):
- Router identifies multi-step/diagnostic queries and routes to Agent tier
- Agent tier runs on frontier models (more expensive, higher capability)
- Agent queries have relaxed latency SLO (8s vs 2s for simple)
- Agent tier auto-scales independently (bursty, GPU-hungry)

WHY NOT Approach A (Monolithic LLM-First) at this scale:
- Single LLM call per query means EVERY query hits expensive model
- At 9M queries/day, cost difference between model tiers = $100K+/day
- No circuit breaker isolation: one bad prompt crashes everything
- Can't independently scale query types (analytics vs docs vs recommendations)

GOOGLE-SPECIFIC ADVANTAGES:
- Self-hosted Gemini on TPUs for the simple query pipelines (5-10x cheaper)
- Vertex AI pipeline orchestration for production deployment
- Spanner for multi-region strong consistency (no eventual consistency bugs)
- Internal bandwidth between services is free (co-located in Google DCs)
```

---

## TRADE-OFF MATRICES

### Database Trade-offs for This System

| Decision | Option A | Option B | Option C | My Choice & Why |
|----------|----------|----------|----------|-----------------|
| **Vector DB** | Pinecone (managed) | pgvector (on PostgreSQL) | Weaviate (self-hosted) | **Pinecone** for <50 customers (zero-ops, fast); migrate to **Weaviate** at scale (cost control, hybrid search built-in) |
| **Analytics DB** | BigQuery (serverless) | PostgreSQL (managed) | ClickHouse (self-hosted) | **BigQuery** — serverless scales to any query complexity, per-customer datasets isolate tenants, SQL interface matches Text-to-SQL pipeline |
| **Document Store** | S3 + metadata DB | MongoDB (flexible schema) | Elasticsearch (search + store) | **S3 + PostgreSQL metadata** — cheapest for blob storage, PostgreSQL handles the structured metadata and querying |
| **Session Store** | Redis (in-memory) | DynamoDB (managed KV) | PostgreSQL (JSONB) | **Redis** — sessions are small, need sub-ms access, TTL for auto-cleanup |
| **Cache Layer** | Redis (general) | Custom semantic cache | LLM provider cache (Anthropic prefix) | **All three**: Redis for exact-match, semantic cache for similar queries, prefix caching for shared context |

### LLM Model Selection Trade-offs

| Query Type | Fast/Cheap Model | Balanced | Best Quality | My Strategy |
|------------|-----------------|----------|--------------|-------------|
| Simple factual | Haiku ($0.001) | Sonnet ($0.01) | Opus ($0.05) | **Haiku** — answer is in the context, just needs extraction |
| Analytical (SQL) | Sonnet ($0.01) | Sonnet ($0.01) | Opus ($0.05) | **Sonnet** — SQL generation is well-handled, don't need Opus |
| Diagnostic (why) | — | Sonnet ($0.01) | Opus ($0.05) | **Opus** — multi-step reasoning benefits from strongest model |
| Recommendation | — | Sonnet ($0.01) | Opus ($0.05) | **Opus** — stakes are highest, accuracy matters most |

**Cost Impact of Routing:**
```
Without routing (all Opus): 36K queries × $0.05 = $1,800/day
With routing:
- 50% simple (Haiku): 18K × $0.001 = $18
- 30% analytical (Sonnet): 10.8K × $0.01 = $108
- 20% complex (Opus): 7.2K × $0.05 = $360
TOTAL: $486/day (73% cost reduction!)
```

### Retrieval Strategy Trade-offs

| Strategy | Precision | Recall | Latency | Cost | Best For |
|----------|-----------|--------|---------|------|----------|
| Dense only (vector) | Medium | High | 50ms | Medium | Semantic similarity, vague queries |
| Sparse only (BM25) | High | Medium | 30ms | Low | Exact keyword matches, entity lookups |
| Hybrid (dense + sparse + RRF) | High | High | 100ms | Medium | Production systems (best overall quality) |
| Multi-query (query expansion) | Highest | Highest | 300ms | High | Complex queries where recall is critical |
| Agentic (iterative retrieval) | Highest | Highest | 2-5s | Very High | Multi-hop reasoning, rare for this use case |

**My choice: Hybrid (dense + sparse + RRF) with multi-query for low-confidence results.**
- Default: hybrid retrieval (100ms, good enough for 85% of queries)
- Fallback: if initial retrieval confidence < 0.7, expand to multi-query
- Never use agentic retrieval for standard queries (too slow for interactive use)

---

## DEEP DESIGN DECISIONS WITH TRADE-OFFS

### Decision 1: Text-to-SQL Approach

**Option A: Direct LLM SQL Generation**
```
User question → LLM generates SQL → Execute → Return results
```
- Pro: Simple, flexible
- Con: LLM can generate invalid/dangerous SQL, schema hallucination
- Accuracy: ~75% first-attempt

**Option B: Semantic Layer (dbt metrics, Cube.js)**
```
User question → LLM selects pre-defined metrics → Execute → Return results
```
- Pro: Safe (only pre-defined queries), fast, guaranteed correct SQL
- Con: Limited to predefined metrics, can't handle novel questions
- Accuracy: ~99% (but limited scope)

**Option C: Hybrid — Semantic Layer + Fallback to Generated SQL**
```
User question → Attempt semantic layer match → 
  If match (80% of queries): use pre-defined metric (fast, safe)
  If no match (20%): generate SQL (slower, validated before execution)
```
- Pro: Best of both — safe for common queries, flexible for novel ones
- Con: More complex to build and maintain
- Accuracy: ~95% overall

**MY CHOICE: Option C (Hybrid)**
```
WHY:
- 80% of analytical queries are recurring patterns 
  ("what's my stockout rate?", "top 10 slow-moving SKUs")
- Pre-define these as metrics → instant, safe, cached
- 20% are novel ("compare this month's returns in NE vs SW region 
  for SKUs sourced from supplier X") → need generated SQL
- Generated SQL goes through validation:
  1. Syntax check (parse AST)
  2. Table/column existence check (against schema)
  3. Permission check (tenant can only access their tables)
  4. Cost estimation (reject if full table scan > 1TB)
  5. Read-only enforcement (no INSERT/UPDATE/DELETE/DROP)
  6. Timeout: 30 second max execution
```

### Decision 2: Conversation State Management

**Option A: Stateless (stuff history in context)**
```
Every request sends full conversation history in the prompt.
```
- Pro: Simple, no state to manage, works with any LLM
- Con: Token cost grows with conversation length, context window limits
- Limit: ~20 turns before context is too long

**Option B: Summarized History**
```
After every N turns, summarize conversation into a condensed memory.
Include summary + last 3 turns in each request.
```
- Pro: Bounded token cost, handles long sessions
- Con: Summary loses nuance, LLM may "forget" early context details
- Limit: Effectively unlimited turns

**Option C: Retrieval-Augmented Memory**
```
Store all turns in a searchable index.
For each new query, retrieve the RELEVANT past turns (not all).
```
- Pro: Scales to very long sessions, only includes relevant context
- Con: Most complex to implement, retrieval may miss relevant history
- Limit: Unlimited, with relevance-based inclusion

**MY CHOICE: Option B (Summarized History)**
```
WHY:
- Avg supply chain session is 5-8 turns (not hundreds)
- Summarization handles 95% of cases without complexity of Option C
- Token savings: 3-turn window + summary vs full history
  = ~1,500 tokens vs ~5,000 tokens = 70% savings per turn
- Implement Option C only if user research shows long-session patterns
```

### Decision 3: Chunking Strategy for Supply Chain Documents

| Strategy | Chunk Size | Overlap | Best For | Weakness |
|----------|-----------|---------|----------|----------|
| Fixed-size (512 tokens) | 512 | 50 tokens | Uniform, simple | Cuts mid-sentence, mid-table |
| Recursive character | 200-1000 | 20% | General text | Doesn't respect document structure |
| Semantic (by topic) | Variable | None | Research papers, reports | Expensive (needs LLM), slow |
| Document-structure | Section-level | None | Contracts, SOPs with headers | Requires structure detection |
| Table-aware | Row or full table | None | Invoices, reports with data | Complex implementation |
| Parent-child | Small (retrieval) + large (context) | Parent contains child | Any | 2x storage, but best quality |

**MY CHOICE: Parent-child with table-awareness**
```
WHY (specific to supply chain docs):
- Supply chain documents are HEAVILY tabular (POs, invoices, inventory reports)
- Standard chunking DESTROYS table context (splits row headers from values)
- Parent-child strategy:
  - Child chunks (small, 256 tokens): used for RETRIEVAL (high precision)
  - Parent chunks (large, 1024 tokens): used for CONTEXT (full section)
  - When a child chunk matches → include the parent in LLM context
  - Tables: always keep as one chunk (even if large)

Example:
  Document: "Q2 Inventory Report"
  Parent chunk: Full "Warehouse Utilization" section (800 tokens)
  Child chunks: 
    - "Warehouse A: 87% utilized, 13% available, 45K pallets"
    - "Warehouse B: 62% utilized, 38% available, 28K pallets"
  
  Query: "Which warehouse has capacity?"
  → Child "Warehouse B: 62%..." matches (high relevance)
  → Parent provides full context (all warehouses for comparison)
  → LLM sees the complete picture
```

---

## FAILURE MODE ANALYSIS (Staff-Level Depth)

### Component Failure Matrix
```
┌──────────────────┬───────────────────┬────────────────────────┬──────────────────────┐
│ Component        │ Failure Mode      │ Impact                 │ Mitigation           │
├──────────────────┼───────────────────┼────────────────────────┼──────────────────────┤
│ LLM Provider     │ Rate limited      │ Queries queue up       │ Multi-provider        │
│ (Anthropic/      │                   │ Latency spikes         │ fallback (Anthropic   │
│  OpenAI)         │                   │                        │ → OpenAI → local)     │
│                  │ Total outage      │ System unusable        │ Graceful degradation  │
│                  │                   │                        │ (cached answers only) │
│                  │ Quality degrade   │ Bad answers, halluc.   │ Quality monitoring,   │
│                  │                   │                        │ auto-switch providers │
├──────────────────┼───────────────────┼────────────────────────┼──────────────────────┤
│ Vector DB        │ High latency      │ Slow retrieval         │ Cache hot queries,    │
│ (Pinecone)       │                   │                        │ read replicas         │
│                  │ Index corruption   │ Wrong docs retrieved   │ Periodic consistency  │
│                  │                   │                        │ checks, rebuild index │
│                  │ Outage            │ No RAG capability      │ Fallback to keyword   │
│                  │                   │                        │ search (Elasticsearch)│
├──────────────────┼───────────────────┼────────────────────────┼──────────────────────┤
│ Analytics DB     │ Query timeout     │ No analytical answers  │ Query cost estimator, │
│ (BigQuery)       │                   │                        │ kill expensive queries│
│                  │ Stale data        │ Answers based on old   │ Show "data as of X",  │
│                  │                   │ data                   │ freshness indicator   │
│                  │ Schema change     │ SQL generation breaks  │ Schema versioning,    │
│                  │                   │                        │ automated tests       │
├──────────────────┼───────────────────┼────────────────────────┼──────────────────────┤
│ Query Classifier │ Misclassification │ Wrong pipeline runs,   │ Low-confidence →      │
│                  │                   │ bad/slow answer        │ multi-pipeline fanout │
│                  │ Model drift       │ Accuracy degrades      │ Weekly accuracy audit │
│                  │                   │ over time              │ on labeled samples    │
├──────────────────┼───────────────────┼────────────────────────┼──────────────────────┤
│ Ingestion        │ ERP disconnect    │ Data grows stale       │ Staleness alerting,   │
│ Pipeline         │                   │                        │ warn users in UI      │
│                  │ Schema migration  │ Broken transformations │ Schema evolution      │
│                  │ in source ERP     │                        │ detection + alerts    │
└──────────────────┴───────────────────┴────────────────────────┴──────────────────────┘
```

### Latency Budget Breakdown (Google-Scale Targets)
```
TOTAL BUDGET: p50 <2s (simple), p50 <5s (complex), p99 <8s (all)
SLO: 99.9% of queries complete within 10 seconds

Simple Query "What's our inventory of SKU-789?":
┌──────────────────────────────┬──────────┬──────────┐
│ Step                         │ Time     │ Cumulative│
├──────────────────────────────┼──────────┼──────────┤
│ Global LB + Edge routing     │ 5ms      │ 5ms      │
│ Auth + Tenant resolution     │ 10ms     │ 15ms     │
│ Semantic cache check         │ 15ms     │ 30ms     │
│ Query Classification (local) │ 20ms     │ 50ms     │
│ Vector Search (regional)     │ 40ms     │ 90ms     │
│ Reranking (GPU, batched)     │ 50ms     │ 140ms    │
│ Context Assembly             │ 10ms     │ 150ms    │
│ LLM Generation (Gemini Nano) │ 800ms    │ 950ms    │
│ Citation + Guard Check       │ 100ms    │ 1,050ms  │
│ Response Formatting          │ 10ms     │ 1,060ms  │
├──────────────────────────────┼──────────┼──────────┤
│ TOTAL                        │          │ ~1.1 sec │
│ (With streaming TTFB)        │          │ ~400ms   │
└──────────────────────────────┴──────────┴──────────┘

Complex Query "Why did we stockout last week across all East Coast DCs?":
┌──────────────────────────────┬──────────┬──────────┐
│ Step                         │ Time     │ Cumulative│
├──────────────────────────────┼──────────┼──────────┤
│ Global LB + Edge routing     │ 5ms      │ 5ms      │
│ Auth + Tenant resolution     │ 10ms     │ 15ms     │
│ Query Classification         │ 20ms     │ 35ms     │
│ Query Planning (Gemini Pro)  │ 500ms    │ 535ms    │
│ PARALLEL EXECUTION:          │          │          │
│   Vector Search + Rerank     │ 90ms     │          │
│   SQL Gen + Spanner Query    │ 1,500ms  │          │
│   Knowledge Graph traversal  │ 200ms    │ 1,735ms  │
│ Context Assembly + Fusion    │ 30ms     │ 1,765ms  │
│ LLM Synthesis (Gemini Ultra) │ 3,000ms  │ 4,765ms  │
│ Citation + Safety Check      │ 200ms    │ 4,965ms  │
│ Response Formatting          │ 15ms     │ 4,980ms  │
├──────────────────────────────┼──────────┼──────────┤
│ TOTAL                        │          │ ~5 sec   │
│ (With streaming TTFB)        │          │ ~2 sec   │
└──────────────────────────────┴──────────┴──────────┘

Agent Query "Diagnose the root cause of our 15% cost increase this quarter":
┌──────────────────────────────┬──────────┬──────────┐
│ Step                         │ Time     │ Cumulative│
├──────────────────────────────┼──────────┼──────────┤
│ Auth + Classification        │ 35ms     │ 35ms     │
│ Agent Planning (Gemini Ultra)│ 1,000ms  │ 1,035ms  │
│ Sub-task 1: Cost breakdown   │ 2,000ms  │ 3,035ms  │
│ Sub-task 2: Supplier changes │ 1,500ms  │ (parallel)│
│ Sub-task 3: Volume analysis  │ 1,800ms  │ 3,035ms  │
│ Synthesis + Recommendation   │ 2,500ms  │ 5,535ms  │
│ Verification + Citations     │ 500ms    │ 6,035ms  │
├──────────────────────────────┼──────────┼──────────┤
│ TOTAL                        │          │ ~6 sec   │
│ (With streaming TTFB)        │          │ ~2.5 sec │
└──────────────────────────────┴──────────┴──────────┘

KEY LATENCY OPTIMIZATIONS AT GOOGLE SCALE:
- Streaming everywhere: First token in <500ms even for complex queries
- Speculative execution: Start top-2 likely pipelines in parallel, cancel loser
- Edge caching: Popular queries cached at CDN edge (CloudFlare/Cloud CDN)
- Prefetching: Predict next query based on session context, pre-warm results
- Connection pooling: Persistent gRPC streams to LLM serving (no cold start)
- Regional locality: All data for a tenant co-located in one region
- Batched inference: GPU batching for vector search + reranking (higher throughput)
- KV-cache sharing: Multiple queries from same tenant share KV-cache prefix
```

---

## CAPACITY PLANNING

### Horizontal Scaling Triggers (Google-Scale)
```
SCALE SIGNAL → ACTION:

QPS > 5,000 sustained (1 min) →
  Auto-scale LLM serving pods (GKE HPA + custom metrics)
  Activate overflow to secondary TPU pod slices
  
LLM latency p99 > 8s →
  Shed load: route overflow to cheaper/faster model tier
  Enable request coalescing for semantically similar concurrent queries
  Activate additional TPU pod slice in same region

Vector DB latency p95 > 150ms →
  Add shard replicas (Vertex Matching Engine auto-handles this)
  Warm up cold tenant indices into memory
  Consider index compaction if fragmented

Cross-region latency p50 > 50ms →
  Check inter-region replication lag
  Consider promoting read replica to primary for affected region

Ingestion lag > 15 minutes →
  Scale up Dataflow/CDC workers (auto-scaling with backlog-based metric)
  Alert data platform team
  Temporarily relax freshness SLO and notify affected tenants

Cost per query > $0.012 avg (24h rolling) →
  Investigate model routing distribution (target: 60% small, 20% medium, 10% large)
  Check semantic cache hit rate (target: >75%)
  Audit for query amplification (agent loops, retry storms)

Global error rate > 0.1% (5 min window) →
  Activate circuit breaker for degraded pipeline
  Route traffic to healthy pipelines only
  Page on-call SRE
```

### Capacity Planning Table (Google-Scale Growth)
```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────────┐
│              │ 1K customers │ 5K customers │ 10K customers│ 50K customers    │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────────┤
│ QPS (avg)    │ 100          │ 550          │ 1,100        │ 5,500            │
│ QPS (peak)   │ 1,000        │ 5,500        │ 11,000       │ 55,000           │
│ QPS (burst)  │ 5,000        │ 25,000       │ 50,000       │ 250,000          │
│ Vector store │ 7.5 TB       │ 37 TB        │ 75 TB        │ 375 TB           │
│ Spanner data │ 200 TB       │ 1 PB         │ 2 PB         │ 10 PB            │
│ LLM cost/mo  │ $160K        │ $800K        │ $1.6M        │ $8M              │
│ Infra cost/mo│ $200K        │ $900K        │ $1.9M        │ $9M              │
│ Total cost/mo│ $360K        │ $1.7M        │ $3.5M        │ $17M             │
│ TPU pods     │ 4            │ 8            │ 16           │ 64               │
│ Engineers    │ 20           │ 50           │ 80           │ 200+             │
│ Regions      │ 2            │ 3            │ 3            │ 5+               │
│ Architecture │ Multi-region │ Multi-region │ Multi-region │ Federated global │
│              │ microservices│ platform     │ platform     │ mesh             │
│ Key concern  │ Cost optim.  │ Multi-tenancy│ Org scaling  │ Federated govern.│
│              │              │ isolation    │              │                  │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────────┘

CRITICAL SCALING INFLECTION POINTS:
- 1K→5K: Must move to self-hosted models (API cost becomes prohibitive)
- 5K→10K: Must solve noisy neighbor problem (large tenants impact small ones)
- 10K→50K: Must federate — no single team can own the whole system
  Need platform team + pipeline teams + per-region ops teams
```
