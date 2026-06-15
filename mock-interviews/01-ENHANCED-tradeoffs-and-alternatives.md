# ENHANCED: AI Supply Chain Copilot — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 1 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES

### Query Volume Modeling
```
USERS:
- 100 enterprise customers
- Avg 30 active users per customer = 3,000 total users
- Power users: 20% send 80% of queries
- Avg session: 3 queries in 10 minutes, 4 sessions/day

QUERY MATH:
- Active hours: 8am-6pm per timezone (effectively 16h with global spread)
- Avg: 3,000 users × 12 queries/day = 36,000 queries/day = 0.6 QPS
- Peak (Monday morning, month-end): 10x = 6 QPS
- Burst (all users in one company run reports): 50 QPS for 5 minutes

This is LOW QPS. The bottleneck is NOT throughput — it's:
1. Latency per query (LLM calls are 3-15 seconds)
2. Cost per query ($0.02-0.10 depending on complexity)
3. Concurrent connections to LLM providers (rate limits)
```

### Storage Modeling
```
PER CUSTOMER:
- ERP data (structured): 50M rows × 200 bytes = 10 GB
- Documents (unstructured): 10K docs × 200KB = 2 GB
- Embeddings: 10K docs × 20 chunks × 6KB = 1.2 GB
- Conversation history: 1K sessions × 5 turns × 2KB = 10 MB
- Total per customer: ~14 GB

ALL CUSTOMERS:
- 100 customers × 14 GB = 1.4 TB
- Growth: 20%/year from new data, 30%/year from new customers
- 3-year projection: ~5 TB

VECTOR DB SIZING:
- 100 customers × 200K chunks × 1536-dim float32 = 
  20M vectors × 6KB = 120 GB (fits in memory for fast retrieval)
- With metadata: ~200 GB total vector storage
```

### Cost Modeling
```
LLM COSTS (per query breakdown):
┌──────────────────────────────────────────────────────────────┐
│ Component           │ Tokens    │ Cost      │ Cacheable?     │
├──────────────────────────────────────────────────────────────┤
│ System prompt       │ 1,000     │ ~$0.003   │ YES (prefix)   │
│ Retrieved context   │ 3,000     │ ~$0.009   │ Partially      │
│ Conversation history│ 1,500     │ ~$0.005   │ YES (prefix)   │
│ User query          │ 50        │ ~$0.000   │ NO             │
│ Output generation   │ 500       │ ~$0.008   │ NO             │
├──────────────────────────────────────────────────────────────┤
│ TOTAL per query     │ ~6,000    │ ~$0.025   │               │
│ With prefix caching │           │ ~$0.012   │ 50% savings    │
│ With semantic cache │           │ ~$0.005   │ 80% hit rate   │
└──────────────────────────────────────────────────────────────┘

MONTHLY COST (steady state):
- 36K queries/day × 30 days × $0.012 (with caching) = $13K/month (LLM)
- Infrastructure (compute, vector DB, storage): ~$8K/month
- Total: ~$21K/month
- Revenue per customer: $50K+/year → healthy margin

SCALING CONCERN:
- At 500 customers: $65K/month LLM + $30K infra = $95K/month
- Need: aggressive caching, model routing (cheap model for simple queries)
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

### My Recommendation & Why:
```
START with Approach B (Pipeline Router) because:
1. Matches the 100-customer scale (need reliability, not just capability)
2. Cost-optimizable (critical at $21K/month LLM costs)
3. Debuggable (enterprise customers demand explainability)
4. Team-scalable (each pipeline can be owned independently)

EVOLVE toward Approach C for complex queries by:
- Adding an "agent" pipeline alongside RAG/Analytics/Recommendation
- Route only complex diagnostic queries to the agent pipeline
- Keep simple queries on fast pipelines (80% of traffic)

This gives the best of both worlds: fast/cheap for simple queries,
capable/expensive for complex queries.
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

### Latency Budget Breakdown
```
TOTAL BUDGET: 10 seconds (simple), 15 seconds (complex)

Simple Query "What's our inventory of SKU-789?":
┌──────────────────────────────┬──────────┬──────────┐
│ Step                         │ Time     │ Cumulative│
├──────────────────────────────┼──────────┼──────────┤
│ API Gateway + Auth           │ 20ms     │ 20ms     │
│ Query Classification         │ 50ms     │ 70ms     │
│ Vector Search (retrieval)    │ 80ms     │ 150ms    │
│ Reranking                    │ 100ms    │ 250ms    │
│ Context Assembly             │ 30ms     │ 280ms    │
│ LLM Generation (Haiku)       │ 1,500ms  │ 1,780ms  │
│ Citation Verification        │ 200ms    │ 1,980ms  │
│ Response Formatting          │ 20ms     │ 2,000ms  │
├──────────────────────────────┼──────────┼──────────┤
│ TOTAL                        │          │ ~2 sec   │
└──────────────────────────────┴──────────┴──────────┘

Complex Query "Why did we stockout last week?":
┌──────────────────────────────┬──────────┬──────────┐
│ Step                         │ Time     │ Cumulative│
├──────────────────────────────┼──────────┼──────────┤
│ API Gateway + Auth           │ 20ms     │ 20ms     │
│ Query Classification         │ 50ms     │ 70ms     │
│ PARALLEL:                    │          │          │
│   Vector Search + Rerank     │ 180ms    │          │
│   SQL Generation + Execution │ 3,000ms  │ 3,070ms  │
│ Context Assembly             │ 50ms     │ 3,120ms  │
│ LLM Synthesis (Opus)         │ 8,000ms  │ 11,120ms │
│ Citation Verification        │ 500ms    │ 11,620ms │
│ Response Formatting          │ 30ms     │ 11,650ms │
├──────────────────────────────┼──────────┼──────────┤
│ TOTAL                        │          │ ~12 sec  │
└──────────────────────────────┴──────────┴──────────┘

OPTIMIZATION LEVERS:
- Streaming: Send first tokens to user while still generating (perceived latency drops 60%)
- Prefetching: While user is typing, pre-classify and pre-retrieve
- Caching: Repeated queries hit cache in <100ms
- Parallel: Run retrieval + SQL simultaneously (saves 3s on complex queries)
```

---

## CAPACITY PLANNING

### Horizontal Scaling Triggers
```
SCALE SIGNAL → ACTION:

Queries > 10 QPS sustained (5 min) →
  Scale up orchestration service pods (K8s HPA)
  
LLM latency p99 > 20s →
  Activate secondary LLM provider
  Enable request coalescing for similar queries

Vector DB latency p95 > 200ms →
  Add read replica
  Increase cache size for hot queries

Ingestion lag > 4 hours →
  Scale up CDC workers
  Alert data team (potential source system issue)

Cost per query > $0.05 avg (7-day rolling) →
  Investigate: classifier pushing too many queries to expensive model?
  Tune confidence thresholds for model routing

Human review queue > 500 items →
  Investigate: accuracy degradation?
  Retrain classifier / update retrieval index
```

### Capacity Planning Table
```
┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│              │ 50 customers │ 200 customers│ 500 customers│ 1000 customers│
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ QPS (avg)    │ 0.3          │ 1.2          │ 3            │ 6            │
│ QPS (peak)   │ 3            │ 12           │ 30           │ 60           │
│ Vector DB    │ 120 GB       │ 500 GB       │ 1.2 TB       │ 2.5 TB       │
│ LLM cost/mo  │ $7K          │ $28K         │ $65K         │ $130K        │
│ Infra cost/mo│ $4K          │ $15K         │ $35K         │ $70K         │
│ Engineers    │ 3-4          │ 6-8          │ 10-15        │ 20+          │
│ Architecture │ Single-region│ Single-region│ Multi-region │ Multi-region │
│              │ monolith     │ microservices│ microservices│ platform     │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```
