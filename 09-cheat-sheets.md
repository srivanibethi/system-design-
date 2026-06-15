# Quick Reference Cheat Sheets

## Cheat Sheet 1: The 5-Minute Opening

Use this template to start ANY system design question:

```
"Before I dive in, let me make sure I understand the scope.

FUNCTIONAL:
- The system needs to [core verb 1], [core verb 2], and [core verb 3]
- The primary users are [who]
- The key workflow is [happy path in 1 sentence]

NON-FUNCTIONAL:
- Scale: [users/QPS/data volume]
- Latency: [target]
- Availability: [target]
- Consistency: [strong/eventual and where]

ASSUMPTIONS I'm making:
- [Assumption 1]
- [Assumption 2]

Does this align with what you have in mind?"
```

---

## Cheat Sheet 2: Back-of-Envelope Numbers

### Quick Conversions
```
1 day = 86,400 seconds ≈ 100K seconds
1M requests/day ≈ 12 QPS
1B requests/day ≈ 12K QPS
1 request/sec for a month ≈ 2.5M requests

Power of 2:
2^10 = 1K      2^20 = 1M      2^30 = 1B      2^40 = 1T
```

### Storage Estimates
```
1 character = 1 byte (ASCII)
1 typical DB row = 100 bytes - 1 KB
1 JSON document = 1-10 KB
1 image (thumbnail) = 50 KB
1 image (full) = 300 KB - 2 MB
1 embedding (1536 dim, float32) = 6 KB
1 minute audio = 1 MB
1 minute video (720p) = 50 MB
```

### QPS Capacity (single machine)
```
Web server (Nginx): 50K-100K concurrent connections
Application server: 1K-10K QPS (depends on logic complexity)
PostgreSQL: 10K simple QPS, 2K complex QPS
Redis: 100K+ QPS
Elasticsearch: 1K-10K search QPS
LLM API call: 1-30 seconds (variable)
```

### Network & Latency
```
Same datacenter:     0.5 ms
Same region:         1-5 ms  
Cross-region (US):   40-80 ms
Cross-continent:     100-200 ms
DNS lookup:          20-120 ms
TCP handshake:       ~RTT
TLS handshake:       2× RTT
```

---

## Cheat Sheet 3: Technology Selection Matrix

### "When should I use...?"

| Scenario | Choose | Not |
|----------|--------|-----|
| Need transactions + complex queries | PostgreSQL | MongoDB |
| Simple key-value, ultra-fast | Redis / DynamoDB | PostgreSQL |
| High write volume, wide column | Cassandra / ScyllaDB | MySQL |
| Full-text search | Elasticsearch | Doing LIKE queries in SQL |
| Time-series metrics | TimescaleDB / InfluxDB | PostgreSQL |
| Vector similarity | pgvector (small) / Pinecone (large) | Elasticsearch (hacky) |
| Analytics/OLAP | BigQuery / ClickHouse | PostgreSQL |
| Global consistency | Spanner / CockroachDB | Cassandra |
| Event streaming | Kafka | RabbitMQ (for streaming) |
| Task queue | SQS / RabbitMQ / Celery | Kafka (overkill) |
| Caching | Redis (structured) / Memcached (simple) | PostgreSQL with UNLOGGED |
| File/blob storage | S3 / GCS | Database BLOBs |
| Real-time sync | Firebase / Supabase Realtime | Polling |
| Service mesh | Envoy/Istio | Doing it in application code |

---

## Cheat Sheet 4: Common Architectures (Draw These Fast)

### Microservices with Event Bus
```
[Client] → [API GW] → [Service A] ──→ [Kafka] ←── [Service B]
                    → [Service C] ──→            ←── [Service D]
                                          ↓
                                    [Analytics DB]
```

### ML Serving
```
[Request] → [Feature Store] → [Model Server] → [Post-process] → [Response]
                                    ↑
                           [Model Registry]
                                    ↑
                          [Training Pipeline]
```

### RAG System
```
[Query] → [Embed] → [Vector Search] → [Rerank] → [LLM + Context] → [Answer]
                          ↑
                    [Document Store]
                          ↑
                    [Ingestion Pipeline]
```

### CQRS + Event Sourcing
```
[Commands] → [Command Handler] → [Event Store] → [Projections] → [Read DB]
                                       ↓                              ↑
                                 [Event Bus]                    [Query Handler]
                                       ↓                              ↑
                                 [Side Effects]                [Client Reads]
```

---

## Cheat Sheet 5: Trade-off Discussions

### Framework: "It Depends On..."
Every "should I use X or Y?" answer follows this pattern:
```
"It depends on:
1. [Primary constraint] — if [condition], use X; otherwise Y
2. [Secondary constraint] — X is better when [scenario]
3. [Operational constraint] — Y is easier to operate when [condition]

For THIS system, I'd choose [X] because [specific reasoning tied to requirements]"
```

### Common Trade-offs (have an opinion on each)

| Trade-off | When to choose LEFT | When to choose RIGHT |
|-----------|--------------------|--------------------|
| SQL vs NoSQL | Complex queries, ACID needed | Simple access patterns, massive scale |
| Monolith vs Microservices | Small team, unclear boundaries | Large team, clear domains, independent deploy |
| Sync vs Async | Need immediate response | Can tolerate delay, need decoupling |
| Push vs Pull | Low latency needed, known subscribers | High fan-out, variable consumers |
| Cache vs No Cache | Read-heavy, tolerance for stale | Write-heavy, freshness critical |
| Strong vs Eventual consistency | Correctness critical (money, inventory) | Availability critical (feeds, analytics) |
| Batch vs Stream | Complete data needed, latency OK | Real-time insights needed |
| Build vs Buy | Core differentiator, unique needs | Commodity, not your competitive advantage |
| Thick client vs Thin client | Offline support, rich UX | Simple deployment, server-side control |

---

## Cheat Sheet 6: Failure Mode Analysis

### For EVERY component, ask:
```
1. What happens when this component is DOWN?
   → Failover? Degraded mode? Error to user?

2. What happens when this component is SLOW?
   → Timeout? Circuit breaker? Fallback?

3. What happens when this component gives WRONG answers?
   → Validation? Checksums? Reconciliation?

4. What happens when this component is OVERLOADED?
   → Backpressure? Shedding? Auto-scale?
```

### Failure Handling Cheat Sheet
| Failure Type | Pattern | Implementation |
|-------------|---------|----------------|
| Transient error | Retry with exponential backoff | 3 retries, 1s/2s/4s + jitter |
| Service down | Circuit breaker | Open after 5 failures in 10s, half-open after 30s |
| Overload | Rate limiting + shedding | Token bucket, priority queue |
| Data corruption | Checksums + reconciliation | CRC32 on records, nightly full reconciliation |
| Split brain | Fencing tokens | Monotonic token required for writes |
| Cascading failure | Bulkhead + timeout | Separate thread pools per dependency |

---

## Cheat Sheet 7: AI/ML System Design Quick Reference

### LLM Integration Checklist
```
□ Model selection (capabilities vs cost vs latency)
□ Prompt engineering (version control, testing)
□ Guardrails (input validation, output validation)
□ Caching (semantic cache, prefix cache)
□ Fallback (if LLM fails/slow, what happens?)
□ Cost tracking (per-query cost, budget alerts)
□ Evaluation (automated metrics + human eval)
□ Observability (trace every LLM call: prompt, response, latency, tokens)
□ Privacy (PII in prompts? Data retention? DPA with provider?)
```

### RAG Quality Checklist
```
□ Chunking strategy matches content type
□ Hybrid retrieval (dense + sparse)
□ Reranking before LLM
□ Citation/source attribution in response
□ Handling "no relevant context found" gracefully
□ Freshness: how quickly do new documents become searchable?
□ Evaluation: faithfulness, relevancy, completeness
□ Filtering by metadata (tenant, date range, document type)
```

### ML Model Lifecycle
```
Data → Features → Training → Evaluation → Registry → Serving → Monitoring
                                                                    ↓
                                                              Drift detected?
                                                                    ↓
                                                              Retrain trigger
```

---

## Cheat Sheet 8: Numbers for Supply Chain Context

### Typical Enterprise Scale
```
Fortune 500 company:
- 10K-100K SKUs
- 50-500 warehouses/DCs
- 1K-10K suppliers
- 1M-100M transactions/day
- 5-50 ERP modules in use
- 10-100 TB of historical data
```

### Supply Chain KPIs to Reference
```
Inventory turnover: 4-12x/year (varies by industry)
Order lead time: 1 day (e-commerce) to 12 weeks (manufacturing)
Perfect order rate: 95-99% target
Forecast accuracy: 70-85% at SKU/week level (good)
Stockout rate: <2% target
Carrying cost: 20-30% of inventory value annually
```

### Cost of Getting It Wrong
```
Stockout cost: Lost sale + lost customer lifetime value
  - Retail: $50-500 per incident
  - Manufacturing: $10K-1M per hour of line stoppage

Overstock cost: 20-30% of excess inventory value/year
  - Perishables: 100% loss after expiry

Forecast error impact:
  - 1% improvement in forecast accuracy ≈ 5-10% reduction in safety stock
  - 10% safety stock reduction on $100M inventory = $10M freed capital
```

---

## Cheat Sheet 9: Communication Templates

### Starting the Deep Dive
```
"The most interesting challenge in this system is [X]. 
Let me go deep on that.

The naive approach would be [simple solution], but that fails because [reason].

Instead, I'd propose [better approach]. Here's how it works..."
```

### Handling "What About...?" Questions
```
"Great question. There are a few options:

Option A: [describe] — Pro: [advantage]. Con: [disadvantage].
Option B: [describe] — Pro: [advantage]. Con: [disadvantage].

Given our requirements of [specific requirement], I'd go with [choice] because [quantified reasoning]."
```

### Wrapping Up
```
"Let me summarize the key decisions:

1. [Decision] — chose [X] over [Y] because [requirement]
2. [Decision] — chose [X] because [quantified trade-off]
3. [Decision] — this was the hardest call; [reasoning]

If I had more time, I'd explore:
- [Area 1] — specifically [what aspect]
- [Area 2] — because [reason it matters]

The biggest risk in this design is [risk], which I'd mitigate by [strategy]."
```

---

## Cheat Sheet 10: "What Would You Do in Week 1?"

X loves people who bias toward action. If asked "how would you start building this?":

```
Week 1 Plan:
- Day 1: Talk to 3 potential users. Understand their workflow TODAY.
- Day 2: Map existing data sources. What data exists? What's missing?
- Day 3: Build the simplest possible prototype (hardcoded if needed)
- Day 4: Put prototype in front of a user. Watch them use it.
- Day 5: Write a 1-pager: "Here's what I learned. Here's the bet I'd make."

What I'm optimizing for: LEARNING SPEED, not code quality.
The goal of week 1 is to know which assumptions are wrong.
```
