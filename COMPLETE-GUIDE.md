# 7-Day Study Plan: X (Alphabet Moonshot) Staff SWE System Design

## Role Context
- **Position**: Staff Software Engineer, Early Stage Supply Chain Project
- **Company**: X (x.company) — Alphabet's moonshot factory
- **Focus**: AI-powered supply chain optimization (LLMs, RAG, Foundation Models)
- **Level**: L6/Staff equivalent

## What Makes This Interview Different

X interviews combine Google's technical rigor with moonshot-specific dimensions:
1. **Ambiguity tolerance** — problems are intentionally underspecified; they want to see you scope
2. **10x thinking** — not incremental improvements, but radical rethinking
3. **AI-native architecture** — LLM/ML systems are core, not bolted on
4. **Enterprise context** — integrating with legacy systems (SAP, Oracle ERP)
5. **Staff-level signals** — you drive architecture, mentor, handle cross-cutting concerns

## Daily Schedule

### Day 1: Foundations & Frameworks
- [ ] Read `01-framework-and-methodology.md` (the universal approach)
- [ ] Read `02-distributed-systems-fundamentals.md`
- [ ] Practice: Design a URL shortener (warm-up, apply framework)
- [ ] Practice: Design a rate limiter (capacity estimation focus)

### Day 2: Data Systems & Storage
- [ ] Read `03-data-systems-deep-dive.md`
- [ ] Practice: Design a time-series database for supply chain metrics
- [ ] Practice: Design a distributed cache with consistency guarantees
- [ ] Review trade-offs: SQL vs NoSQL vs NewSQL vs Vector DBs

### Day 3: AI/ML Systems (Critical for this role)
- [ ] Read `04-ai-ml-systems-design.md`
- [ ] Practice: Design a RAG system for enterprise document Q&A
- [ ] Practice: Design an LLM serving platform with guardrails
- [ ] Study: Embedding pipelines, vector search, retrieval strategies

### Day 4: Supply Chain + Enterprise Domain
- [ ] Read `05-supply-chain-ai-systems.md`
- [ ] Practice: Design an AI-powered demand forecasting system
- [ ] Practice: Design a real-time inventory optimization engine
- [ ] Study: ERP integration patterns, data mesh, event-driven architecture

### Day 5: Practice Problems (Full Sessions)
- [ ] Read `06-practice-problems-with-solutions.md`
- [ ] Do 2 full 45-minute mock designs (timer yourself)
- [ ] Focus on: scoping, trade-offs, Staff-level depth
- [ ] Record yourself explaining (or explain to a rubber duck)

### Day 6: Advanced Topics + Moonshot Thinking
- [ ] Read `07-advanced-topics.md`
- [ ] Read `08-moonshot-thinking-guide.md`
- [ ] Practice: Design a system that doesn't exist yet (novel problem)
- [ ] Study: How to handle "I don't know" gracefully at Staff level

### Day 7: Review & Polish
- [ ] Read `09-cheat-sheets.md` (rapid review)
- [ ] Do 1 final full mock (most challenging problem)
- [ ] Review your weak areas from the week
- [ ] Rest, sleep well, trust your preparation

## Staff-Level Evaluation Criteria (What Interviewers Look For)

| Signal | What They Want to See |
|--------|----------------------|
| **Scope** | You drive the conversation, define boundaries, make explicit trade-offs |
| **Depth** | Deep expertise in at least 2-3 areas; can go multiple levels deep on any component |
| **Breadth** | Comfortable across the full stack; understand how pieces connect |
| **Ambiguity** | Comfortable with incomplete information; make reasonable assumptions, state them |
| **Trade-offs** | Never "best" — always "best given constraints X, Y, Z" |
| **Leadership** | Proactively identify risks, propose phased rollouts, consider operational concerns |
| **Communication** | Clear, structured, collaborative; invite discussion |

## Key Differences: L5 vs L6 System Design

| Dimension | L5 (Senior) | L6 (Staff) |
|-----------|-------------|-------------|
| Scope | Single service/component | Multi-service system, cross-cutting concerns |
| Ambiguity | Clarifies requirements | Defines the problem space itself |
| Trade-offs | Identifies them | Quantifies and decides with reasoning |
| Depth | Goes deep when asked | Proactively dives where it matters |
| Operations | Mentions monitoring | Designs for observability, rollout, failure modes |
| Leadership | Executes well | Shapes direction, identifies what to NOT build |


---




---


# System Design Interview Framework

## The 4-Phase Approach (45-50 minutes)

### Phase 1: Requirements & Scoping (5-8 minutes)

**This is where Staff engineers differentiate themselves.**

#### Functional Requirements
Ask: "What does this system need to DO?"
- Core use cases (2-3 max for a 45-min interview)
- User personas and workflows
- Input/output contracts

#### Non-Functional Requirements
Ask: "What qualities must it have?"
- **Scale**: Users, requests/sec, data volume, growth rate
- **Latency**: p50, p95, p99 targets
- **Availability**: 99.9%? 99.99%? What's the cost of downtime?
- **Consistency**: Strong vs eventual? Where does it matter?
- **Durability**: Can we lose data? Recovery time objectives?

#### Constraints & Assumptions
- Budget/team size (startup vs big company)
- Existing infrastructure (cloud provider, tech stack)
- Regulatory requirements (GDPR, SOX for supply chain)
- Timeline (MVP vs production-grade)

**Staff-Level Move**: Don't just ask questions — propose constraints and validate them.
- "I'm going to assume we need 99.9% availability since supply chain disruptions cost $X/hour. Does that align?"
- "Given this is enterprise software, I'll assume we need SOC 2 compliance and audit logging."

#### Back-of-Envelope Calculations
Always estimate early:
```
Users: 10,000 enterprise users
Peak QPS: 10,000 users × 10 requests/hour = ~28 QPS (bursty to 5x = 140 QPS)
Storage: 1M supply chain events/day × 1KB avg = 1GB/day = 365GB/year
LLM calls: 500 queries/hour × $0.01/query = $120/day
```

---

### Phase 2: High-Level Design (10-12 minutes)

#### Start with the API
Define the contract first:
```
POST /api/v1/forecast
  Request: { sku_ids: [], horizon_days: 30, confidence_level: 0.95 }
  Response: { forecasts: [{ sku_id, date, predicted_demand, confidence_interval }] }
```

#### Draw the Architecture
Components to consider:
- **Client Layer**: Web app, mobile, API consumers
- **API Gateway / Load Balancer**: Rate limiting, auth, routing
- **Application Services**: Business logic, orchestration
- **Data Layer**: Databases, caches, message queues
- **AI/ML Layer**: Model serving, feature stores, embedding services
- **Infrastructure**: Monitoring, logging, deployment

#### Data Flow
Trace the happy path end-to-end:
1. User submits request →
2. API gateway authenticates and rate-limits →
3. Service orchestrates data retrieval →
4. ML model generates prediction →
5. Result is cached and returned

**Staff-Level Move**: Identify the 2-3 hardest problems in your design and call them out.
- "The interesting challenges here are: (1) keeping predictions fresh as new data arrives, (2) handling the cold-start problem for new SKUs, and (3) explaining predictions to non-technical users."

---

### Phase 3: Deep Dive (20-25 minutes)

**This is where interviews are won or lost at Staff level.**

Pick 2-3 components to go deep on. Prioritize:
1. The component most critical to the system's success
2. The component with the most interesting trade-offs
3. The component the interviewer shows interest in

#### For each deep dive:
- **Data model**: Schema design, indexing strategy, partitioning
- **Algorithm/approach**: Why this algorithm? What are alternatives?
- **Failure modes**: What happens when this component fails?
- **Scale**: How does this component scale? What's the bottleneck?
- **Trade-offs**: What did you give up? Why is that acceptable?

#### Trade-off Framework
For every decision, articulate:
```
Decision: Use eventual consistency for inventory counts
Why: Strong consistency would require distributed transactions,
     adding 200ms latency and reducing throughput 10x
Trade-off: Users may see stale counts for up to 5 seconds
Mitigation: Show "last updated" timestamp; use strong consistency
           for order placement (where correctness matters)
```

---

### Phase 4: Wrap-Up & Extensions (5-8 minutes)

#### Operational Concerns
- **Monitoring**: What metrics matter? (latency percentiles, error rates, ML model drift)
- **Alerting**: What pages someone at 2am?
- **Deployment**: Canary → staged → full rollout
- **Testing**: How do you test a distributed system?

#### Evolution & Scale
- "If we 10x traffic, what breaks first?"
- "How would we add multi-region support?"
- "What would a v2 look like?"

#### Staff-Level Closing Moves
- Summarize the key trade-offs you made and why
- Identify what you'd investigate further with more time
- Mention organizational/team implications ("This needs a dedicated ML platform team")

---

## Common Pitfalls to Avoid

| Pitfall | Fix |
|---------|-----|
| Jumping into solution immediately | Always start with requirements |
| Designing for infinite scale from day 1 | Start simple, identify scale triggers |
| Only one option considered | Present 2-3 approaches, pick with reasoning |
| Handwaving trade-offs | Quantify: "This adds 50ms but saves $X/month" |
| Ignoring failure modes | Every component can fail; design for it |
| Not involving the interviewer | "I'm thinking about X. Would you like me to go deeper or move on?" |
| Designing a single box | At Staff level, think multi-service, multi-team |

---

## The "Staff Signal" Checklist

Use this as a mental checklist during your interview:

- [ ] I defined the problem before solving it
- [ ] I made explicit assumptions and validated them
- [ ] I did back-of-envelope math
- [ ] I considered at least 2 approaches before picking one
- [ ] I quantified trade-offs (not just "faster" but "200ms faster at cost of...")
- [ ] I identified the hardest problems and went deep on them
- [ ] I considered operational concerns (monitoring, deployment, failure)
- [ ] I discussed how the system evolves over time
- [ ] I communicated clearly and structured my thinking
- [ ] I drove the conversation while remaining collaborative


---


# Distributed Systems Fundamentals

## Core Concepts You Must Know Cold

### CAP Theorem
You can only guarantee 2 of 3:
- **Consistency**: Every read gets the most recent write
- **Availability**: Every request gets a response (not an error)
- **Partition Tolerance**: System works despite network partitions

**Reality**: Partitions WILL happen, so you're really choosing between CP and AP.
- **CP** (Consistency + Partition Tolerance): Banking, inventory at checkout → use strong consensus (Raft, Paxos)
- **AP** (Availability + Partition Tolerance): Social feeds, analytics, caches → use eventual consistency (CRDTs, last-write-wins)

**Staff-level nuance**: CAP is about the ENTIRE system, but you can make different choices for different subsystems. Order placement = CP. Product recommendations = AP.

### PACELC Theorem (extends CAP)
When there IS a Partition: choose Availability or Consistency
Else (normal operation): choose Latency or Consistency

Example: DynamoDB is PA/EL — during partitions choose availability, during normal operation choose low latency (eventual consistency). You can opt into strong reads at higher latency.

---

### Consistency Models

| Model | Guarantee | Use Case | Example |
|-------|-----------|----------|---------|
| Strong/Linearizable | Reads always see latest write | Financial transactions | Spanner |
| Sequential | All nodes see same order | Distributed locks | ZooKeeper |
| Causal | Causally related ops seen in order | Chat messages | MongoDB sessions |
| Eventual | All replicas converge eventually | Analytics, DNS | DynamoDB, Cassandra |

---

### Consensus Algorithms

#### Raft (most interview-relevant)
- Leader election + log replication
- Leader sends AppendEntries to followers
- Committed when majority acknowledges
- Leader failure → new election (term increment)
- **Key insight**: Reads can be stale unless you do leader lease or read quorums

#### Paxos
- More general but harder to understand
- Single-decree Paxos: agree on one value
- Multi-Paxos: chained for log replication
- Mention awareness but use Raft for explanations

---

## Scalability Patterns

### Horizontal Scaling

#### Load Balancing
- **L4 (TCP)**: Fast, no content inspection (AWS NLB)
- **L7 (HTTP)**: Content-based routing, SSL termination (Nginx, Envoy)
- **Algorithms**: Round-robin, least-connections, consistent hashing, weighted

#### Sharding/Partitioning
- **Hash-based**: hash(key) % N → uniform distribution, hard to rebalance
- **Range-based**: key ranges → good for range queries, risk of hot spots
- **Consistent hashing**: minimal redistribution on node add/remove
- **Directory-based**: lookup table → flexible but single point of failure

**Staff insight**: Always discuss rebalancing strategy. How do you add a shard without downtime?

#### Replication
- **Leader-follower**: One writer, many readers. Simple but leader is bottleneck.
- **Multi-leader**: Multiple writers. Conflict resolution needed (LWW, vector clocks, CRDTs).
- **Leaderless**: Quorum reads/writes (R + W > N). Dynamo-style.

---

### Caching

#### Strategies
| Pattern | How It Works | Best For |
|---------|-------------|----------|
| Cache-aside | App checks cache, falls back to DB, fills cache | Read-heavy, tolerance for stale |
| Write-through | Write to cache AND DB simultaneously | Consistency-critical |
| Write-behind | Write to cache, async flush to DB | Write-heavy (risk: data loss) |
| Read-through | Cache auto-populates from DB on miss | Simplify app logic |

#### Cache Invalidation (the hard problem)
- **TTL**: Simple, eventual staleness bounded
- **Event-driven**: DB change → publish event → invalidate cache
- **Version-based**: Include version in cache key; new version = cache miss

#### Distributed Cache Architecture
- Redis Cluster: hash slots (16384), master-replica per slot
- Memcached: Client-side consistent hashing, no replication

**Staff insight**: Cache stampede mitigation — request coalescing, probabilistic early expiration, locking.

---

### Message Queues & Event Streaming

#### When to Use What
| Tool | Use Case | Semantics |
|------|----------|-----------|
| Kafka | Event streaming, log aggregation | At-least-once, ordered within partition |
| RabbitMQ | Task queues, RPC | At-most-once or at-least-once |
| SQS | Simple cloud queues | At-least-once, no ordering guarantee |
| Pub/Sub (GCP) | Event fanout | At-least-once, ordering optional |

#### Key Kafka Concepts
- **Partitions**: Unit of parallelism. More partitions = more throughput.
- **Consumer groups**: Each partition consumed by one consumer in group.
- **Offset management**: Consumer tracks position. Replay by resetting offset.
- **Retention**: Time or size based. Infinite retention = event sourcing.
- **Compacted topics**: Keep latest value per key. Good for state sync.

---

## Reliability Patterns

### Failure Handling
- **Retry with exponential backoff + jitter**: Prevent thundering herd
- **Circuit breaker**: Stop calling failing service; fail fast
- **Bulkhead**: Isolate failures; don't let one bad dependency take everything down
- **Timeout**: Always set timeouts. No timeout = infinite hang.
- **Idempotency**: Design operations to be safely retried (idempotency keys)

### High Availability
- **Redundancy**: No single points of failure
- **Health checks**: Liveness vs readiness probes
- **Graceful degradation**: Serve stale data vs error; reduce features vs crash
- **Chaos engineering**: Proactively inject failures to find weaknesses

### Data Durability
- **WAL (Write-Ahead Log)**: Write to log before applying. Recover from crashes.
- **Replication factor**: 3 replicas across availability zones
- **Backups**: Point-in-time recovery, cross-region backup
- **Checksums**: Detect silent data corruption

---

## Networking & Communication

### Synchronous
- **REST**: Simple, widely understood, HTTP-based
- **gRPC**: Binary protocol (protobuf), streaming, efficient. Good for service-to-service.
- **GraphQL**: Client-specified queries. Good for varied client needs.

### Asynchronous
- **Message queues**: Decouple producer/consumer
- **Webhooks**: Push notification on events
- **Server-Sent Events (SSE)**: Server pushes to client (unidirectional)
- **WebSockets**: Bidirectional real-time communication

**Staff insight**: For service mesh, mention Envoy sidecars, mTLS, service discovery (Consul/Kubernetes DNS), and observability (distributed tracing with OpenTelemetry).

---

## Numbers Every Engineer Should Know

```
L1 cache reference:                  1 ns
L2 cache reference:                  4 ns
Main memory reference:              100 ns
SSD random read:                 16,000 ns  (16 μs)
HDD seek:                     4,000,000 ns  (4 ms)
Send packet SF → NYC:        40,000,000 ns  (40 ms)
Send packet SF → London:    100,000,000 ns  (100 ms)

Throughput:
- SSD sequential read: 4 GB/s
- Network (10 Gbps): 1.25 GB/s
- HDD sequential: 200 MB/s

Daily volumes:
- 1M requests/day ≈ 12 QPS
- 100M requests/day ≈ 1,200 QPS
- 1B requests/day ≈ 12,000 QPS

Storage:
- 1 char = 1 byte (ASCII) or 4 bytes (UTF-8 worst case)
- 1 tweet ≈ 300 bytes
- 1 image (compressed) ≈ 300 KB
- 1 min video (720p) ≈ 50 MB
```

---

## Database Selection Guide

| Need | Choose | Why |
|------|--------|-----|
| ACID transactions, complex queries | PostgreSQL | Battle-tested relational, JSONB support |
| Global scale, strong consistency | Spanner / CockroachDB | Distributed SQL with TrueTime/hybrid clocks |
| High write throughput, flexible schema | Cassandra / ScyllaDB | LSM-tree, tunable consistency |
| Document store, developer velocity | MongoDB | Flexible schema, good query language |
| Key-value, ultra-low latency | Redis / DynamoDB | In-memory or SSD-optimized |
| Time-series data | TimescaleDB / InfluxDB | Optimized for time-indexed writes/queries |
| Graph relationships | Neo4j / Neptune | Traversal queries, relationship-first |
| Vector similarity search | Pinecone / Weaviate / pgvector | ANN algorithms (HNSW, IVF) |
| Full-text search | Elasticsearch / OpenSearch | Inverted index, relevance scoring |
| Analytics/OLAP | BigQuery / ClickHouse | Columnar, MPP, SQL on petabytes |


---


# Data Systems Deep Dive

## Storage Engines

### LSM Trees (Log-Structured Merge Trees)
**Used by**: Cassandra, RocksDB, LevelDB, HBase

**How it works**:
1. Writes go to in-memory buffer (memtable)
2. When full, flush to immutable sorted SSTable on disk
3. Background compaction merges SSTables

**Trade-offs**:
- Write-optimized (sequential I/O)
- Reads may check multiple SSTables (use bloom filters)
- Space amplification during compaction
- Compaction strategies: Size-tiered (write-optimized) vs Leveled (read-optimized)

### B-Trees
**Used by**: PostgreSQL, MySQL, most RDBMS

**How it works**:
- Self-balancing tree of fixed-size pages (4-16KB)
- Updates in place with WAL for crash safety
- O(log n) for reads and writes

**Trade-offs**:
- Read-optimized (single path to leaf)
- Write amplification (rewrite entire page for one row change)
- Good for range queries (leaves are linked)

### When to Choose
- **High write throughput, simple reads**: LSM (Cassandra, DynamoDB)
- **Complex queries, transactions, mixed workload**: B-Tree (PostgreSQL)
- **Time-series (append-heavy)**: LSM with time-based partitioning

---

## Schema Design Patterns

### Relational (Star Schema for Analytics)
```sql
-- Fact table (events/transactions)
CREATE TABLE supply_chain_events (
    event_id BIGINT PRIMARY KEY,
    timestamp TIMESTAMPTZ NOT NULL,
    sku_id INT REFERENCES dim_products(sku_id),
    warehouse_id INT REFERENCES dim_warehouses(warehouse_id),
    event_type VARCHAR(50),
    quantity INT,
    unit_cost DECIMAL(10,2)
);

-- Dimension tables
CREATE TABLE dim_products (
    sku_id INT PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(100),
    supplier_id INT
);
```

### Document Model (Supply Chain Context)
```json
{
  "order_id": "ORD-123456",
  "status": "in_transit",
  "items": [
    { "sku": "SKU-789", "qty": 100, "unit_price": 45.00 }
  ],
  "shipment": {
    "carrier": "FedEx",
    "tracking": "FX123",
    "eta": "2026-06-20T14:00:00Z",
    "route": ["warehouse_SFO", "hub_LAX", "dest_NYC"]
  },
  "metadata": {
    "created_by": "system_auto_reorder",
    "priority": "high",
    "sla_deadline": "2026-06-21T00:00:00Z"
  }
}
```

### Wide-Column (Time-Series Events)
```
Row Key: warehouse_id#date#event_id
Column Families:
  - metrics: { quantity, cost, lead_time }
  - metadata: { source_system, confidence_score }
  - ai: { prediction, model_version, explanation }
```

---

## Indexing Strategies

### Primary Index Types
- **B-Tree index**: Default. Good for equality and range queries.
- **Hash index**: O(1) equality lookups. No range queries.
- **GiST/GIN**: Full-text search, JSONB, geometric data (PostgreSQL).
- **BRIN**: Block Range Index. Excellent for naturally ordered data (timestamps).

### Composite Indexes
```sql
-- Order matters! Leftmost prefix rule.
CREATE INDEX idx_events ON supply_chain_events (warehouse_id, timestamp, event_type);

-- This index supports:
-- WHERE warehouse_id = X                    ✓
-- WHERE warehouse_id = X AND timestamp > Y  ✓
-- WHERE timestamp > Y                       ✗ (can't skip first column)
```

### Partitioning Strategies
- **Range partitioning**: By date (time-series), by region
- **Hash partitioning**: By user_id, order_id (uniform distribution)
- **List partitioning**: By category, status, region

```sql
-- PostgreSQL range partitioning
CREATE TABLE events (
    id BIGSERIAL,
    timestamp TIMESTAMPTZ,
    data JSONB
) PARTITION BY RANGE (timestamp);

CREATE TABLE events_2026_q1 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
```

---

## Distributed Database Patterns

### Single-Leader Replication
```
Client → Leader → [WAL] → Follower 1
                        → Follower 2
                        → Follower 3

Reads: Any replica (eventual) or leader-only (strong)
Writes: Leader only
Failover: Promote follower to leader (manual or automatic)
```

**Problems**: Split brain, replication lag, failover data loss

### Multi-Leader Replication
Use case: Multi-region deployments where each region needs local writes

**Conflict resolution**:
- Last-write-wins (LWW): Simple but lossy
- Custom merge functions: Application-specific
- CRDTs: Mathematically guaranteed convergence

### Leaderless (Dynamo-style)
```
Write to W nodes, Read from R nodes
Consistency guarantee: R + W > N

Example: N=3, W=2, R=2
- Every read overlaps with writes → guaranteed fresh data
- But: slower reads/writes, conflict resolution needed
```

---

## Data Pipeline Architectures

### Lambda Architecture
```
                    ┌─────────────────────────┐
Raw Data ──→ Batch Layer (MapReduce/Spark) ──→ Batch View
    │                                              ↓
    └──→ Speed Layer (Flink/Kafka Streams) ──→ Serving Layer ←── Queries
              (real-time, approximate)
```
- Batch: Complete, accurate, high latency
- Speed: Recent data, low latency, approximate
- Merge at serving layer

### Kappa Architecture (preferred for new systems)
```
Raw Data → Kafka → Stream Processor (Flink) → Serving DB → Queries
                                              ↓
                                         Materialized Views
```
- Single processing path (streaming)
- Reprocess by replaying Kafka from beginning
- Simpler to maintain than Lambda

### Modern Data Stack for AI/ML
```
Sources → Ingestion → Lake/Warehouse → Feature Store → Model Serving
  │         │              │                │              │
  ERP    Fivetran/      BigQuery/         Feast/        Vertex AI/
  IoT    Airbyte        Snowflake         Tecton        SageMaker
  APIs   Kafka          Databricks                      Custom
```

---

## Transaction Patterns

### SAGA Pattern (Distributed Transactions)
When you can't use 2PC (which you usually can't at scale):

```
Order Service → Payment Service → Inventory Service → Shipping Service
     │                │                  │                   │
     └── Compensate ←─┘── Compensate ←──┘── Compensate ←───┘
         (cancel order)   (refund)        (restock)        (cancel shipment)
```

**Choreography**: Each service publishes events, others react
**Orchestration**: Central coordinator manages the saga steps

**Staff insight**: Always discuss compensation logic. What if step 3 fails after steps 1-2 succeeded?

### Outbox Pattern
Ensure exactly-once event publishing without distributed transactions:
```
BEGIN TRANSACTION
  INSERT INTO orders (...)
  INSERT INTO outbox (event_type, payload, published=false)
COMMIT

-- Separate process polls outbox table and publishes to Kafka
-- Mark published=true after successful publish
-- Consumer is idempotent (handles duplicates)
```

---

## Change Data Capture (CDC)

### Architecture
```
PostgreSQL → Debezium → Kafka → Consumers (search index, cache, analytics)
  (WAL)      (CDC connector)
```

### Use Cases for Supply Chain
- Sync ERP changes to AI systems in real-time
- Keep search indexes up-to-date
- Feed ML feature stores with latest data
- Cross-system consistency without tight coupling

### Implementation Options
- **Log-based** (Debezium): Reads DB WAL. No schema changes needed. Most reliable.
- **Trigger-based**: DB triggers write to audit table. Overhead on writes.
- **Polling**: Periodic queries with timestamp column. Simple but not real-time.

---

## Capacity Estimation Template

### Storage
```
Daily events: 10M events × 500 bytes/event = 5 GB/day
Monthly: 150 GB
Yearly: 1.8 TB
With 3x replication: 5.4 TB
With indexes (1.5x): ~8 TB total

Growth: 2x per year → plan for 16 TB in year 2
```

### Throughput
```
Writes: 10M events/day = 116 events/sec avg, 580/sec peak (5x)
Reads: 50M queries/day = 580 QPS avg, 2900 QPS peak

Single PostgreSQL: ~10K simple QPS, ~2K complex QPS
→ Need read replicas or sharding at this scale? Probably not yet.
→ Cache hit rate of 90% means 290 QPS hit DB → comfortable on single primary
```

### Network
```
API response size: 2 KB average
Peak bandwidth: 2900 QPS × 2 KB = 5.8 MB/s = 46 Mbps
→ Well within single server capacity (10 Gbps)
```


---


# AI/ML Systems Design (Critical for X Role)

## RAG System Architecture

### What is RAG?
Retrieval-Augmented Generation: Enhance LLM responses with relevant context retrieved from a knowledge base. Critical for enterprise AI where the LLM needs access to private, up-to-date data.

### Full RAG Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                     INGESTION PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│ Source Docs → Chunking → Embedding → Vector DB              │
│ (PDFs, ERP,   (semantic/   (text-embedding-   (Pinecone,    │
│  Confluence,   fixed-size/   3-large,           Weaviate,    │
│  Slack)        recursive)    Cohere embed)      pgvector)    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     QUERY PIPELINE                            │
├─────────────────────────────────────────────────────────────┤
│ User Query → Query Transform → Retrieval → Reranking → LLM │
│              (expansion,       (vector +    (cross-encoder,  │
│               HyDE,            keyword      Cohere rerank)   │
│               decomposition)   hybrid)                       │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

#### Chunking Strategy
| Strategy | Chunk Size | Best For |
|----------|-----------|----------|
| Fixed-size | 256-512 tokens | Simple, predictable |
| Recursive/semantic | Variable | Documents with structure |
| Sentence-level | 1-3 sentences | FAQ, definitions |
| Document-level | Full doc | Short documents, metadata-rich |
| Sliding window | Overlap 20% | Context continuity |

**Staff insight**: Chunking is the #1 factor in RAG quality. Too small = lost context. Too large = diluted relevance. Always include surrounding context (parent-child chunking).

#### Retrieval Strategies
- **Dense retrieval**: Embedding similarity (cosine/dot product). Good for semantic meaning.
- **Sparse retrieval**: BM25/TF-IDF. Good for exact keyword matches.
- **Hybrid**: Combine both with Reciprocal Rank Fusion (RRF). Best in practice.
- **Multi-vector**: ColBERT-style token-level matching. Higher quality, more compute.

```python
# Hybrid search with RRF
def hybrid_search(query, k=10):
    dense_results = vector_db.search(embed(query), top_k=k*2)
    sparse_results = elasticsearch.search(query, top_k=k*2)
    return reciprocal_rank_fusion(dense_results, sparse_results, k=k)

def reciprocal_rank_fusion(results_lists, k=60):
    scores = defaultdict(float)
    for results in results_lists:
        for rank, doc in enumerate(results):
            scores[doc.id] += 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])
```

#### Reranking
After initial retrieval (cheap, broad), apply expensive cross-encoder reranking:
```
Retrieved 50 docs (fast, vector search)
    → Rerank with cross-encoder → Top 5 (high quality, expensive)
        → Feed to LLM context window
```

#### Advanced RAG Patterns
- **Query decomposition**: Break complex queries into sub-queries, retrieve for each
- **HyDE (Hypothetical Document Embeddings)**: Generate hypothetical answer, embed that for retrieval
- **Self-RAG**: LLM decides when to retrieve and self-grades relevance
- **Corrective RAG (CRAG)**: If retrieval quality is low, fall back to web search or different strategy
- **Agentic RAG**: LLM uses tools to iteratively retrieve and reason

---

## LLM Serving Architecture

### Production LLM System
```
┌──────────────────────────────────────────────────────┐
│                    API GATEWAY                         │
│  Rate limiting, auth, request routing, cost tracking  │
├──────────────────────────────────────────────────────┤
│                  ORCHESTRATION LAYER                   │
│  Prompt management, chain execution, tool routing     │
├──────────────────────────────────────────────────────┤
│              ┌──────────┐  ┌───────────┐             │
│              │ LLM Pool │  │ Guard-    │             │
│              │ (routing) │  │ rails     │             │
│              └──────────┘  └───────────┘             │
│                   │                │                   │
│         ┌─────────┼────────────────┤                  │
│         │         │                │                   │
│    ┌────▼───┐ ┌───▼────┐ ┌───────▼──────┐           │
│    │GPT-4/  │ │Claude  │ │Fine-tuned    │           │
│    │Gemini  │ │        │ │local model   │           │
│    └────────┘ └────────┘ └──────────────┘           │
├──────────────────────────────────────────────────────┤
│              EVALUATION & OBSERVABILITY                │
│  Tracing, quality metrics, cost tracking, A/B tests   │
└──────────────────────────────────────────────────────┘
```

### Key Components

#### Prompt Management
```python
# Version-controlled prompts with A/B testing
class PromptRegistry:
    def get_prompt(self, name, version=None, context=None):
        template = self.store.get(name, version or "latest")
        # Dynamic prompt selection based on context
        if context.complexity == "high":
            return template.variants["detailed"]
        return template.variants["concise"]
```

#### Guardrails
- **Input guardrails**: PII detection, prompt injection detection, topic filtering
- **Output guardrails**: Hallucination detection, format validation, toxicity checks
- **Structural guardrails**: Output schema enforcement (JSON mode, function calling)

#### Model Router
Route to appropriate model based on:
- Task complexity (simple → Haiku, complex → Opus)
- Latency requirements (streaming vs batch)
- Cost constraints
- Quality requirements

```python
def route_request(request):
    if request.requires_reasoning:
        return "claude-opus-4"
    elif request.high_throughput:
        return "claude-haiku-4"
    elif request.domain_specific:
        return "fine_tuned_model_v3"
    return "claude-sonnet-4"
```

#### Caching for LLMs
- **Semantic caching**: Embed the query, check if similar query was recently answered
- **Exact caching**: Hash of (prompt + params) → cached response
- **Prefix caching**: Cache KV states for shared prompt prefixes (Anthropic supports this)

---

## Embedding Pipeline Design

### Architecture
```
Raw Content → Preprocessing → Chunking → Embedding Model → Vector DB
                  │                           │                  │
            (cleaning,              (batch inference,     (HNSW index,
             normalization,          GPU cluster,          metadata
             language detection)     rate limiting)        filtering)
```

### Scaling Considerations
- **Batch vs real-time**: Batch for bulk ingestion, real-time for new documents
- **Model selection**: text-embedding-3-large (OpenAI), Cohere embed-v3, open-source (BGE, E5)
- **Dimensionality**: 768-3072 dimensions. More = better quality, more storage/compute
- **Quantization**: Reduce dimensions or quantize (int8) for 4x storage savings with <5% quality loss

### Vector Database Selection
| Database | Type | Best For |
|----------|------|----------|
| Pinecone | Managed | Zero-ops, auto-scaling |
| Weaviate | Open-source | Hybrid search, multi-modal |
| Qdrant | Open-source | Filtering + vector search |
| pgvector | Extension | Already on PostgreSQL, moderate scale |
| Milvus | Open-source | Billion-scale, GPU support |
| Chroma | Lightweight | Prototyping, small-medium scale |

### HNSW (Hierarchical Navigable Small World) — How Vector Search Works
- Build navigable graph layers (coarse → fine)
- Search: start at top layer, greedily traverse to nearest neighbors, descend
- Parameters: `ef_construction` (build quality), `M` (connections per node), `ef_search` (query quality)
- Trade-off: Higher params = better recall, more memory, slower build

---

## ML Model Serving

### Serving Patterns
| Pattern | Latency | Throughput | Use Case |
|---------|---------|------------|----------|
| Online (real-time) | <100ms | Low-medium | User-facing predictions |
| Batch | Hours | Very high | Nightly reports, bulk scoring |
| Near-real-time | Seconds | Medium | Streaming predictions |
| Edge | <10ms | Low | IoT, mobile |

### Online Serving Architecture
```
Request → Feature Store (online) → Model Server → Post-processing → Response
              │                        │
         (pre-computed             (TensorRT/
          features, <10ms)          Triton/
                                    vLLM)
```

### Feature Store
**Why**: Consistency between training and serving. Avoid training-serving skew.

```
┌─────────────────────────────────────────┐
│           FEATURE STORE                  │
├──────────────────┬──────────────────────┤
│  Offline Store   │   Online Store        │
│  (training data) │   (serving features)  │
│  Parquet/BQ      │   Redis/DynamoDB      │
│  High latency    │   <10ms latency       │
└──────────────────┴──────────────────────┘
```

**Key features**:
- Point-in-time correctness (no future leakage in training)
- Feature versioning and lineage
- Monitoring for data drift and staleness

---

## LLM Evaluation & Observability

### Evaluation Framework
```
Offline Eval                    Online Eval
────────────                    ───────────
- Benchmark datasets            - A/B tests
- Human preference              - User feedback (👍/👎)
- LLM-as-judge                  - Task completion rate
- RAG metrics (RAGAS):          - Hallucination rate
  - Faithfulness                - Latency percentiles
  - Answer relevancy            - Cost per query
  - Context precision
  - Context recall
```

### Key Metrics for Production LLM Systems
- **Latency**: Time to first token (TTFT), tokens/second, total response time
- **Quality**: Answer accuracy, hallucination rate, format compliance
- **Cost**: $/query, $/token, cache hit rate
- **Reliability**: Error rate, timeout rate, retry rate
- **Safety**: Guardrail trigger rate, PII leak rate

### Observability Stack
```
Application → OpenTelemetry → Traces/Metrics/Logs
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              Trace Store      Metric Store     Log Store
              (Jaeger)         (Prometheus)     (Loki)
                    │               │               │
                    └───────────────┼───────────────┘
                                    │
                              Grafana Dashboards
                              + PagerDuty Alerts
```

---

## AI Agent Architecture

### Multi-Agent Systems (Relevant for supply chain)
```
┌────────────────────────────────────────┐
│           ORCHESTRATOR AGENT            │
│  (decomposes task, routes to specialists)│
├────────────┬──────────┬────────────────┤
│ Demand     │ Inventory│ Logistics      │
│ Forecaster │ Optimizer│ Router         │
│ Agent      │ Agent    │ Agent          │
├────────────┼──────────┼────────────────┤
│ Tools:     │ Tools:   │ Tools:         │
│ - SQL query│ - ERP API│ - Maps API     │
│ - Stats    │ - Cost   │ - Carrier APIs │
│ - Charts   │   calc   │ - Route solver │
└────────────┴──────────┴────────────────┘
```

### Agent Design Patterns
- **ReAct**: Reason → Act → Observe → Repeat
- **Plan-and-Execute**: Plan all steps first, then execute sequentially
- **Reflexion**: Self-evaluate after execution, retry with learnings
- **Multi-agent debate**: Multiple agents argue, consensus emerges

### Tool Use Architecture
```python
tools = [
    Tool(name="query_inventory", fn=query_inventory_db,
         description="Get current inventory levels for SKUs"),
    Tool(name="get_demand_forecast", fn=call_forecast_model,
         description="Get demand prediction for next N days"),
    Tool(name="calculate_reorder_point", fn=reorder_calc,
         description="Calculate optimal reorder point given lead time and demand"),
]
```

---

## Handling Hallucination in Enterprise Systems

### Prevention Strategies
1. **Grounding**: Always provide source documents; instruct model to cite
2. **Constrained generation**: JSON mode, function calling, enum responses
3. **Retrieval verification**: Check if answer is supported by retrieved context
4. **Confidence scoring**: Model self-reports confidence; low confidence → human review
5. **Chain-of-verification**: Generate answer → generate verification questions → check consistency

### Detection Methods
- **NLI-based**: Use natural language inference model to check if response is entailed by context
- **LLM-as-judge**: Second LLM verifies claims against source
- **Factual consistency scores**: Compare entities/claims between source and response
- **Citation verification**: Check that cited passages actually support the claim

### Staff Insight: Enterprise AI Reliability
For supply chain decisions (where hallucinations cost real money):
- Never let LLM make autonomous decisions above $X threshold
- Always show provenance (which documents informed this answer)
- Human-in-the-loop for high-stakes actions
- Audit trail for regulatory compliance
- Graceful degradation: if AI is uncertain, fall back to rule-based system


---


# Supply Chain AI Systems Design

## Why This Matters for X

X's moonshot is applying AI to reduce waste, fragility, and inefficiency in global supply chains. The system design interview will likely include at least one problem that touches:
- Enterprise integration (SAP, Oracle ERP)
- Real-time decision making under uncertainty
- AI-augmented human workflows
- Massive-scale optimization problems

---

## Supply Chain System Architecture

### End-to-End AI-Powered Supply Chain Platform
```
┌──────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                  │
├────────────┬──────────────┬──────────────┬───────────────────────┤
│ ERP Systems│ IoT/Sensors  │ External     │ Historical            │
│ (SAP, Oracle)│ (warehouse, │ (weather,    │ (orders, shipments,   │
│            │  transport)  │  market,     │  demand history)      │
│            │              │  suppliers)  │                       │
└─────┬──────┴──────┬───────┴──────┬───────┴───────────┬───────────┘
      │             │              │                   │
      ▼             ▼              ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                   INTEGRATION LAYER                                │
│  CDC (Debezium) │ Event Bus (Kafka) │ API Gateway │ Batch ETL    │
└──────────────────────────────────────┬───────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    AI/ML PLATFORM                                  │
├──────────────┬───────────────┬───────────────┬───────────────────┤
│ Feature Store│ Model Registry│ Training      │ Serving           │
│ (online +    │ (versioned    │ Pipeline      │ Infrastructure    │
│  offline)    │  models)      │ (Vertex AI)   │ (online + batch)  │
└──────────────┴───────────────┴───────────────┴───────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                                 │
├─────────────┬────────────────┬────────────────┬──────────────────┤
│ Demand      │ Inventory      │ Logistics      │ Supplier         │
│ Forecasting │ Optimization   │ Routing        │ Risk             │
│ Service     │ Service        │ Service        │ Assessment       │
└─────────────┴────────────────┴────────────────┴──────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     USER LAYER                                     │
│  Decision Support UI │ Alerts & Recommendations │ Natural Language │
│  (dashboards, what-if)│ (proactive notifications)│ Interface (LLM) │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key System Designs

### 1. Demand Forecasting System

#### Requirements
- Predict demand for 100K+ SKUs across 500+ locations
- Multiple time horizons: next day, next week, next quarter
- Handle seasonality, promotions, external events
- Confidence intervals, not just point estimates
- Explainability for business users

#### Architecture
```
┌─────────────────────────────────────────────────┐
│              DATA COLLECTION                      │
│ Historical sales │ Promotions │ Weather │ Events │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│            FEATURE ENGINEERING                    │
│ Lag features │ Rolling stats │ Calendar features │
│ Price elasticity │ Cannibalization │ External    │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│              MODEL ENSEMBLE                       │
│ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│ │Statistical│ │ ML-based │ │ Foundation Model │ │
│ │(Prophet,  │ │(LightGBM,│ │ (TimesFM,       │ │
│ │ ETS, ARIMA│ │ XGBoost) │ │  Chronos, Lag-  │ │
│ │           │ │          │ │  Llama)          │ │
│ └──────────┘ └──────────┘ └──────────────────┘ │
│         ↓            ↓              ↓            │
│              ENSEMBLE / META-LEARNER             │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│           SERVING & MONITORING                   │
│ Batch predictions (nightly) + on-demand refresh  │
│ Drift detection │ Accuracy tracking │ Alerts     │
└─────────────────────────────────────────────────┘
```

#### Key Trade-offs
| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Model per SKU vs global | Individual models (accurate) | Global model (generalizes) | Hierarchical: global + SKU-specific adjustments |
| Batch vs streaming | Nightly batch (simpler) | Real-time updates (fresher) | Batch + event-triggered refresh for high-impact SKUs |
| Point vs probabilistic | Single value (simple) | Distribution (actionable) | Quantile regression — always give confidence intervals |

---

### 2. Inventory Optimization Engine

#### The Core Problem
Decide **when** and **how much** to order for each SKU at each location, minimizing:
- Holding costs (capital tied up, warehousing)
- Stockout costs (lost sales, customer churn)
- Ordering costs (per-order fixed costs, bulk discounts)

#### Architecture
```
┌─────────────────────────────────────────────────┐
│              INPUTS                               │
│ Demand forecast │ Lead times │ Costs │ Constraints│
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│         OPTIMIZATION ENGINE                      │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Safety Stock Calculator                    │   │
│  │ SS = z × σ_demand × √(lead_time)         │   │
│  │ + z × demand_avg × σ_lead_time           │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Reorder Point = avg_demand × lead_time    │   │
│  │              + safety_stock               │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Order Quantity (EOQ variant)              │   │
│  │ Consider: bulk discounts, shelf life,     │   │
│  │          warehouse capacity, cash flow    │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│        CONSTRAINT SOLVER                         │
│ Budget limits │ Warehouse capacity │ MOQs        │
│ Supplier constraints │ Transportation limits     │
│ → Mixed Integer Programming (OR-Tools / Gurobi) │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│        DECISION SUPPORT                          │
│ Recommended orders │ What-if simulation          │
│ Exception alerts │ Auto-execute below threshold  │
└─────────────────────────────────────────────────┘
```

#### Scaling Challenges
- 100K SKUs × 500 locations = 50M optimization problems
- Interdependencies (substitution, complementary products)
- Multi-echelon (warehouse → DC → store)
- Solution: Decompose into independent subproblems where possible, use heuristics + local optimization

---

### 3. Real-Time Supply Chain Visibility

#### Requirements
- Track shipments across global supply chain in real-time
- Predict delays before they happen (ETA prediction)
- Alert relevant stakeholders automatically
- Handle data from heterogeneous sources (IoT, carrier APIs, customs)

#### Architecture
```
Data Sources                   Processing              Serving
────────────                   ──────────              ───────
Carrier APIs ─┐
GPS/IoT ──────┤               ┌──────────┐
Customs data ─┼──→ Kafka ──→  │  Flink   │──→ Materialized Views
Port systems ─┤               │ (stream  │      (current state)
Weather APIs ─┘               │ process) │          │
                              └──────────┘          ▼
                                   │         ┌──────────────┐
                                   │         │ ETA Prediction│
                                   │         │ Model (ML)    │
                                   │         └──────┬───────┘
                                   │                │
                                   ▼                ▼
                              ┌──────────┐   ┌──────────────┐
                              │ Alert    │   │ Visibility   │
                              │ Engine   │   │ Dashboard    │
                              └──────────┘   └──────────────┘
```

#### ETA Prediction Model
Features:
- Historical transit times for this route/carrier
- Current location and velocity
- Weather conditions along route
- Port congestion data
- Day of week, holidays, peak season

Model: Gradient-boosted trees (LightGBM) or survival analysis

---

## Enterprise Integration Patterns

### ERP Integration (SAP/Oracle)

#### Challenge
Legacy ERP systems are:
- Not cloud-native (often on-premise)
- Batch-oriented (nightly jobs, not real-time)
- Complex schemas (thousands of tables, custom fields)
- Change-resistant (any modification = expensive consulting)

#### Integration Approaches

| Approach | Latency | Complexity | Risk |
|----------|---------|------------|------|
| Direct DB read (CDC) | Seconds | High | Schema coupling |
| API layer (SAP OData, Oracle REST) | Seconds | Medium | Rate limits, auth |
| Middleware (MuleSoft, Dell Boomi) | Seconds-minutes | Low | Vendor lock-in |
| File-based (SFTP/S3) | Hours | Low | Very late data |
| Event-driven (SAP Event Mesh) | Sub-second | Medium | Modern SAP only |

#### Recommended Architecture: Anti-Corruption Layer
```
┌──────────┐     ┌─────────────────┐     ┌──────────────┐
│   ERP    │────→│ Anti-Corruption │────→│  AI Platform │
│ (SAP)    │     │ Layer (ACL)      │     │              │
└──────────┘     └─────────────────┘     └──────────────┘
                        │
                 - Translates ERP schemas to clean domain model
                 - Handles ERP quirks (null encoding, date formats)
                 - Provides stable API even if ERP schema changes
                 - Caches for performance
                 - Validates data quality
```

**Staff insight**: Always propose an Anti-Corruption Layer when integrating with legacy systems. It's a DDD pattern that protects your system from the complexity of the external system.

### Data Quality in Enterprise Context
- **Completeness**: Missing values in ERP exports (30-40% common in enterprise data)
- **Timeliness**: Batch jobs may be delayed; stale data = bad predictions
- **Accuracy**: Manual entry errors, duplicate records
- **Consistency**: Same entity different in different systems (product IDs)

#### Solution: Data Quality Pipeline
```
Raw Data → Schema Validation → Business Rules → Statistical Checks → Clean Data
              (types, nulls)    (valid ranges,    (outlier detection,    (ready for
                                 referential       distribution shift)    ML/analytics)
                                 integrity)
```

---

## Event-Driven Architecture for Supply Chain

### Why Events?
Supply chains are inherently event-driven:
- Order placed, shipped, delivered, returned
- Inventory received, allocated, depleted
- Forecast updated, anomaly detected
- Supplier lead time changed, price changed

### Event Schema Design
```json
{
  "event_id": "evt_abc123",
  "event_type": "inventory.level_changed",
  "timestamp": "2026-06-14T10:30:00Z",
  "source": "warehouse_management_system",
  "entity": {
    "type": "sku_location",
    "id": "SKU-789_WH-SFO"
  },
  "payload": {
    "previous_quantity": 500,
    "new_quantity": 450,
    "change_reason": "order_fulfilled",
    "reference_id": "ORD-456"
  },
  "metadata": {
    "correlation_id": "corr_xyz",
    "schema_version": "2.1"
  }
}
```

### CQRS Pattern (Command Query Responsibility Segregation)
```
Commands (writes)                    Queries (reads)
─────────────────                    ──────────────
Place Order ──→ Command Handler      Dashboard ←── Read Model (materialized view)
                    │                 API ←────── Optimized for queries
                    ▼                              (denormalized, pre-computed)
               Event Store
                    │
                    ▼
              Event Handlers
              (update read models,
               trigger workflows,
               notify services)
```

**When to use CQRS**: Different read and write patterns (supply chain: writes are transactional events, reads are analytical aggregations).

---

## Multi-Tenant Architecture (Enterprise SaaS)

### Isolation Models
| Model | Isolation | Cost | Complexity | Use Case |
|-------|-----------|------|------------|----------|
| Silo (separate DB per tenant) | Highest | Highest | Medium | Regulated industries |
| Bridge (shared DB, separate schemas) | Medium | Medium | Medium | Most enterprise SaaS |
| Pool (shared everything, tenant_id column) | Lowest | Lowest | High (noisy neighbor) | Small tenants, high scale |

### For Supply Chain AI Platform
Recommended: **Bridge model** with per-tenant encryption keys
- Each enterprise customer gets isolated data
- Shared compute infrastructure (cost-efficient)
- Tenant-specific ML models (trained on their data only)
- Cross-tenant models for cold-start (anonymized, federated)

### Noisy Neighbor Prevention
- Resource quotas per tenant (CPU, memory, API calls)
- Separate queues per tenant for batch jobs
- Priority tiers (enterprise → standard → free)
- Circuit breakers to prevent one tenant's load from affecting others


---


# Practice Problems with Model Solutions

## Problem 1: Design an AI-Powered Supply Chain Copilot

### Problem Statement
"Design a system where supply chain managers can ask natural language questions about their operations and get actionable answers grounded in their company's data."

Examples:
- "Why did we stockout on SKU-789 last week?"
- "What's the risk of our top supplier being disrupted?"
- "Recommend reorder quantities for Q3 given the promotion calendar"

---

### Solution Walkthrough

#### Phase 1: Requirements (5 min)

**Functional**:
- Natural language Q&A over supply chain data
- Answers grounded in company's actual data (not hallucinated)
- Support for analytical queries (aggregations, trends)
- Support for recommendations (optimization-backed)
- Audit trail for all AI-generated recommendations
- Multi-turn conversation with context

**Non-Functional**:
- Latency: <10s for simple queries, <30s for complex analysis
- Accuracy: 95%+ factual correctness (enterprise requirement)
- Scale: 1000 concurrent users, 10K queries/hour
- Security: SOC 2, data isolation per customer
- Availability: 99.9%

**Back-of-envelope**:
```
10K queries/hour = ~3 QPS average
Assume 20% need real-time data fetch, 80% answerable from cached/indexed data
LLM cost: 3 QPS × $0.02/query = $0.06/sec = $5,200/day
→ Need caching, routing to cheaper models for simple queries
```

#### Phase 2: High-Level Design (10 min)

```
User → Web UI → API Gateway → Query Router
                                    │
                    ┌───────────────┼───────────────────┐
                    ▼               ▼                   ▼
              Simple Q&A      Analytical           Recommendation
              (RAG path)      (SQL path)           (Optimization path)
                    │               │                   │
                    ▼               ▼                   ▼
              Vector DB +     Text-to-SQL +        Optimization
              Doc Store       Data Warehouse       Engine + ML
                    │               │                   │
                    └───────────────┼───────────────────┘
                                    ▼
                              LLM Synthesis
                              (with citations)
                                    │
                                    ▼
                              Response + Sources
                              + Confidence Score
```

**Key insight**: Not all queries are the same. Route to specialized pipelines.

#### Phase 3: Deep Dives (25 min)

**Deep Dive 1: Query Classification & Routing**

```python
QUERY_TYPES = {
    "factual": "RAG pipeline — retrieve from indexed documents",
    "analytical": "Text-to-SQL — query structured data",
    "predictive": "ML model serving — forecasts, risk scores",
    "recommendation": "Optimization engine — what-if scenarios",
    "conversational": "Context-aware follow-up handling"
}

# Two-stage routing:
# 1. Fast classifier (fine-tuned small model) determines query type
# 2. Route to appropriate pipeline
```

**Deep Dive 2: RAG Pipeline for Enterprise Data**

Sources to index:
- ERP master data (products, suppliers, warehouses)
- Historical orders and shipments
- Internal documents (SOPs, contracts, supplier reviews)
- Previous AI interactions (learn from corrections)

Chunking strategy:
- Structured data: Convert to natural language summaries per entity
- Documents: Recursive chunking with 512-token chunks, 20% overlap
- Time-series: Summarize trends into text descriptions

Retrieval:
```
Query → Classify → [If factual] → Hybrid Search (vector + keyword)
                                        → Rerank (cross-encoder)
                                        → Top-5 contexts
                                        → LLM generates answer with citations
```

**Deep Dive 3: Text-to-SQL for Analytical Queries**

```
"Why did we stockout on SKU-789 last week?"
    ↓
Query Understanding: intent=root_cause, entity=SKU-789, time=last_week, event=stockout
    ↓
Schema Selection: inventory_levels, demand_history, supply_orders, stockout_events
    ↓
SQL Generation (LLM + few-shot examples):
    SELECT date, inventory_level, daily_demand, 
           supply_expected, supply_actual
    FROM inventory_daily
    WHERE sku_id = 'SKU-789' AND date BETWEEN '2026-06-07' AND '2026-06-14'
    ↓
Execution + Result
    ↓
LLM Synthesis: "SKU-789 stocked out on June 10 because:
    1. Demand spiked 3x (promotion started June 8)
    2. Supplier delivery was 2 days late (arrived June 12 vs expected June 10)
    3. Safety stock was set for normal demand, not promotional levels"
```

Safety: SQL is validated (read-only, no DDL/DML), query cost estimated before execution, timeout enforced.

#### Phase 4: Wrap-up (5 min)

**Failure modes**:
- LLM hallucination → Citation required, confidence scoring, human review for low confidence
- Slow queries → Timeout with partial answer, async notification for complex queries
- Data freshness → Show "data as of" timestamp, real-time for critical metrics

**Evolution**:
- V1: Q&A only (RAG + Text-to-SQL)
- V2: Add recommendations (optimization engine)
- V3: Proactive alerts ("I notice a potential stockout in 3 days")
- V4: Multi-agent with autonomous actions (place orders, adjust forecasts)

---

## Problem 2: Design a Real-Time Demand Sensing System

### Problem Statement
"Design a system that detects demand changes in real-time and adjusts forecasts within minutes, not days."

---

### Solution Walkthrough

#### Requirements
- Ingest real-time POS (point-of-sale) data from 10,000 stores
- Detect demand signal changes within 15 minutes
- Distinguish signal from noise (real trend vs random fluctuation)
- Update forecasts and trigger downstream actions (reorder, reallocation)
- Handle 1M events/minute at peak (Black Friday)

#### High-Level Architecture
```
POS Systems → Kafka → Stream Processor → Anomaly Detection → Decision Engine
(10K stores)          (Flink)            (statistical +       (update forecast,
                                          ML-based)            alert, reorder)
```

#### Key Components

**1. Ingestion Layer**
```
POS events: {store_id, sku_id, timestamp, quantity, price}
Volume: 1M events/min peak = 16K events/sec
→ Kafka with 100 partitions (partition by store_id for ordering)
→ 3x replication, 7-day retention
```

**2. Real-Time Aggregation (Flink)**
```
Tumbling windows: 15-minute aggregations per (sku, location)
- Current demand vs forecast
- Current demand vs same-time-last-week
- Velocity (demand acceleration/deceleration)

Sliding windows: 1-hour moving averages for smoothing
```

**3. Anomaly Detection**
Two-tier approach:
- **Statistical**: Z-score against forecast (fast, interpretable)
  - If |actual - forecast| > 3σ → anomaly
- **ML-based**: Isolation Forest trained on historical patterns
  - Catches complex patterns (correlated anomalies across SKUs)

**4. Decision Engine**
```python
def handle_demand_signal(signal):
    if signal.severity == "critical":  # >5σ deviation
        # Immediate action
        trigger_emergency_reorder(signal.sku, signal.location)
        alert_supply_chain_manager(signal)
    elif signal.severity == "warning":  # 3-5σ deviation
        # Update forecast, recommend action
        update_short_term_forecast(signal)
        queue_recommendation(signal)
    else:  # 2-3σ
        # Log, monitor, update model inputs
        update_demand_features(signal)
```

#### Trade-offs
| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Window size | 15 min | Balance between responsiveness and noise filtering |
| False positive tolerance | 5% | Better to over-alert than miss a real signal |
| Forecast update frequency | Per-signal | Near-real-time for critical SKUs |
| Autonomy level | Recommend, don't auto-execute | Enterprise customers want human approval |

---

## Problem 3: Design a Multi-Modal Document Understanding System

### Problem Statement
"Design a system that ingests supply chain documents (invoices, BOLs, purchase orders, contracts) in any format (PDF, images, emails) and extracts structured data for downstream systems."

---

### Solution Walkthrough

#### Requirements
- Process 100K documents/day across multiple formats
- Extract structured fields (PO number, line items, dates, quantities)
- 99% accuracy for critical fields (amounts, dates)
- Human-in-the-loop for low-confidence extractions
- Learn from corrections over time

#### Architecture
```
Document Ingestion → Classification → Extraction → Validation → Output
      │                    │              │             │           │
 S3/Email/API      Doc type model    Multi-modal    Business    Structured
                   (invoice vs PO    LLM + layout   rules +     JSON to ERP
                    vs BOL)          aware model    confidence
                                                   scoring
```

#### Deep Dive: Extraction Pipeline
```
PDF/Image → OCR/Layout Analysis → Structured Extraction → Post-Processing
              │                        │                       │
         (Document AI /          (GPT-4V / Claude       (entity resolution,
          Azure Form              with structured         cross-reference
          Recognizer)             output + schema)        with master data)
```

#### Handling Uncertainty
```python
def process_document(doc):
    extraction = extract_fields(doc)
    
    for field in extraction.fields:
        if field.confidence > 0.95:
            field.status = "auto_approved"
        elif field.confidence > 0.80:
            field.status = "needs_review"
            queue_for_human_review(field)
        else:
            field.status = "manual_entry"
            route_to_data_entry_team(field)
    
    # Active learning: human corrections feed back into model
    if any_corrections_made(extraction):
        add_to_training_queue(doc, corrections)
```

#### Scale Design
- **Batch processing**: Queue documents in SQS, process with auto-scaling workers
- **Prioritization**: Urgent documents (time-sensitive POs) processed first
- **Caching**: Same template documents → cache extraction schema
- **Model versioning**: Shadow mode for new models (run both, compare, switch when better)

---

## Problem 4: Design Google-Scale URL Shortener (Classic Warm-Up)

### 10-Minute Speed Run

**Requirements**: Shorten URLs, redirect, analytics, 100M URLs created/day

**Math**:
- 100M/day = 1200 QPS writes, 10x reads = 12K QPS reads
- 7-char base62 = 62^7 = 3.5 trillion unique URLs (sufficient)
- Storage: 100M × 500 bytes = 50GB/day = 18TB/year

**Architecture**:
```
Write: Client → API → ID Generator → DB (write) → Cache (write-through)
Read:  Client → Cache → DB (on miss) → 301 Redirect
```

**Key decisions**:
- ID generation: Pre-generated ID pool (avoid counter bottleneck) or hash-based with collision check
- Storage: DynamoDB or Cassandra (simple KV, high write throughput)
- Cache: Redis cluster, LRU eviction (hot URLs stay cached)
- Analytics: Async event to Kafka → Flink → analytics DB

**Staff-level additions**:
- Custom short URLs (vanity): separate namespace, uniqueness check
- Expiration: TTL on records, batch cleanup
- Abuse prevention: rate limiting, malicious URL detection
- Multi-region: geo-routing, eventual consistency acceptable

---

## Problem 5: Design a Notification/Alerting System for Supply Chain Events

### Problem Statement
"Design a system that monitors supply chain KPIs and sends intelligent alerts to the right person at the right time through the right channel."

---

### Solution Walkthrough

#### Requirements
- Monitor 1000+ metrics across the supply chain
- Intelligent alerting (not just threshold-based — detect anomalies, predict issues)
- Route alerts to correct person based on role, responsibility, schedule
- Multi-channel: Slack, email, SMS, push notification, in-app
- Alert fatigue mitigation (grouping, dedup, escalation)
- 10K enterprise users, 500 organizations

#### Architecture
```
Metrics Stream → Alert Engine → Routing → Delivery → Acknowledgment
                     │              │         │            │
              (threshold +     (on-call    (multi-     (escalation
               anomaly +        schedule,   channel,    if no ack
               predictive)      role-based) retry)      in N min)
```

#### Alert Engine Design
```python
class AlertRule:
    # Static threshold
    SIMPLE = "inventory < safety_stock"
    
    # Anomaly detection
    ANOMALY = "demand deviates >3σ from forecast"
    
    # Predictive
    PREDICTIVE = "stockout probability > 80% in next 7 days"
    
    # Composite
    COMPOSITE = "supplier_delay AND low_inventory AND high_demand"

class AlertProcessor:
    def evaluate(self, metric_event):
        triggered_rules = self.match_rules(metric_event)
        
        for rule in triggered_rules:
            # Dedup: same alert within window → suppress
            if self.is_duplicate(rule, metric_event):
                continue
            
            # Group: related alerts → single notification
            group = self.find_or_create_group(rule, metric_event)
            group.add(metric_event)
            
            # Delay for grouping window (30s), then fire
            if group.ready_to_fire():
                self.route_and_deliver(group)
```

#### Alert Fatigue Mitigation
1. **Deduplication**: Same alert within 15-min window → single notification with count
2. **Grouping**: Related alerts (same root cause) → single summary
3. **Severity escalation**: Info → Warning → Critical (escalate if unresolved)
4. **Smart scheduling**: Non-urgent alerts batched to daily digest
5. **Snooze & mute**: User can snooze specific alerts temporarily
6. **Auto-resolve**: Alert clears when metric returns to normal

---

## Problem 6: Design a Feature Store for ML Models

### Problem Statement
"Design a feature store that serves both batch training pipelines and real-time model inference for a supply chain AI platform."

---

### Key Points for a Strong Answer

#### Dual serving requirement
```
                    ┌─────────────────────┐
Training Jobs ────→ │   Offline Store      │ (BigQuery/S3 Parquet)
                    │   - Historical data   │
                    │   - Point-in-time     │
                    │     correct joins     │
                    └─────────────────────┘
                    
                    ┌─────────────────────┐
Model Serving ────→ │   Online Store       │ (Redis/DynamoDB)
                    │   - Latest features   │
                    │   - <10ms latency     │
                    │   - Precomputed       │
                    └─────────────────────┘
```

#### Critical guarantee: Training-Serving Consistency
The #1 cause of ML production failures is training-serving skew — model trained on features computed one way but served features computed differently.

Solution:
- Single feature definition (code) used for both offline computation and online serving
- Backfill mechanism: same transformation, different time ranges
- Monitoring: compare online feature distributions to training distributions

#### Feature Types
```python
# Entity features (slowly changing)
supplier_reliability_score  # Updated daily
warehouse_capacity         # Updated on config change

# Event features (real-time)
orders_last_1h            # Rolling window
inventory_velocity        # Rate of change

# On-demand features (computed at request time)
days_until_stockout       # forecast / current_velocity
```

---

## Tips for All Practice Problems

### Time Management
- 5-8 min: Requirements + math
- 10-12 min: High-level design
- 20-25 min: Deep dives (2-3 components)
- 5-8 min: Wrap-up, extensions, operational concerns

### Staff-Level Signals to Hit
1. **You drive the conversation** — don't wait to be asked
2. **Quantify everything** — "fast" → "p99 < 200ms"
3. **Name the hard problems** — show you see what others miss
4. **Consider failure** — every component WILL fail
5. **Think operationally** — who gets paged? How do you deploy?
6. **Show evolution** — V1 (MVP) → V2 (scale) → V3 (advanced)


---


# Advanced Topics for Staff-Level Interviews

## 1. Distributed Consensus & Coordination

### When You Need Strong Consistency
- Inventory at checkout (can't oversell)
- Financial transactions
- Leader election
- Distributed locks
- Configuration management

### Google Spanner's Approach
- **TrueTime**: GPS + atomic clocks give globally synchronized timestamps
- **External consistency**: Transactions ordered by real time
- **Read-write transactions**: 2PC within a Paxos group
- **Read-only transactions**: Snapshot reads at a consistent timestamp, no locks

**Interview insight**: If asked "how would you do globally consistent transactions?", mention Spanner's approach — commit timestamps from TrueTime guarantee ordering without cross-region coordination for reads.

### Distributed Locking Patterns
```
Option 1: Redis SETNX (simple but imperfect)
  SET lock_key unique_value NX PX 30000
  - Problem: Clock drift, split brain if Redis fails

Option 2: Redlock (Martin Kleppmann criticized this)
  - Acquire on N/2+1 nodes
  - Problem: Still relies on timing assumptions

Option 3: ZooKeeper/etcd (correct)
  - Sequential ephemeral nodes for fairness
  - Session heartbeats for liveness
  - Watches for notification

Option 4: Chubby/Google approach
  - Consensus-based lock service
  - Sequencer tokens prevent stale locks
```

**Staff insight**: Mention "fencing tokens" — even with distributed locks, the protected resource should verify the lock is still valid. Locks can expire while the holder is GC-paused.

---

## 2. Large-Scale Data Processing

### MapReduce → Dataflow Model → Modern Streaming

**Evolution**:
- MapReduce (2004): Batch, simple, slow
- Spark (2012): In-memory, DAG-based, faster batch
- Flink (2014): True streaming, exactly-once, event-time processing
- Dataflow/Beam (2015): Unified batch + streaming API

### Stream Processing Guarantees
| Guarantee | Meaning | How |
|-----------|---------|-----|
| At-most-once | May lose events | Fire and forget |
| At-least-once | May duplicate | Retry on failure |
| Exactly-once | No loss, no duplication | Checkpointing + idempotent sinks |

### Windowing (Critical for supply chain real-time)
```
Tumbling Window:  [---5min---][---5min---][---5min---]
Sliding Window:   [---5min---]
                    [---5min---]
                      [---5min---]
Session Window:   [---events---]  gap  [---events---]
```

### Watermarks & Late Data
- **Watermark**: "I believe all events up to time T have arrived"
- **Late data handling**: 
  - Drop (simple)
  - Allow lateness window (recompute affected windows)
  - Side output (process separately)

**Supply chain example**: POS data from stores may arrive late (network issues). Use allowed lateness of 1 hour for demand aggregation; recompute affected forecasts.

---

## 3. Multi-Region Architecture

### Patterns

#### Active-Passive (DR)
```
Primary (us-west) ← All traffic
Secondary (us-east) ← Async replication, standby
```
- Simple, but wasteful (secondary idle) and slow failover

#### Active-Active (Low latency)
```
us-west: Handles AMER users
eu-west: Handles EMEA users  
ap-east: Handles APAC users
```
- Each region handles reads + writes for its users
- Cross-region replication for global data
- **Hard problem**: Conflict resolution for concurrent writes

#### Follow-the-Sun (Supply chain specific)
```
During APAC business hours: APAC region handles supply chain ops
During EMEA business hours: EMEA region takes over
During AMER business hours: AMER region active
```
- Supply chain is inherently time-zone sensitive
- Optimize for where operations are happening NOW

### Global Load Balancing
- **DNS-based** (Route 53): Geo-routing, health checks
- **Anycast**: Same IP, routed to nearest healthy endpoint
- **CDN**: Static assets + API caching at edge (CloudFlare, Fastly)

### Cross-Region Data Consistency
- **Global tables** (DynamoDB): Automatic multi-region replication, last-writer-wins
- **Spanner**: Strong consistency globally (but higher latency)
- **CockroachDB**: Configurable locality for latency-sensitive data
- **Application-level**: Region-affinity for user data, eventual sync for reference data

---

## 4. API Design at Scale

### REST Best Practices (Enterprise Context)
```
# Resource-oriented URLs
GET    /api/v1/warehouses/{id}/inventory
POST   /api/v1/orders
PATCH  /api/v1/orders/{id}/status
DELETE /api/v1/forecasts/{id}

# Filtering, pagination, sorting
GET /api/v1/inventory?warehouse=WH-01&sku=SKU-789&page=2&limit=50&sort=-quantity

# Versioning
/api/v1/...  (URL versioning — simple, explicit)
Accept: application/vnd.company.v2+json  (header versioning — cleaner)
```

### Rate Limiting Algorithms
| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| Token bucket | Tokens refill at fixed rate; each request costs a token | Most common, allows bursts |
| Sliding window log | Track timestamps of all requests | Accurate but memory-heavy |
| Sliding window counter | Weighted combination of current + previous window | Good balance |
| Fixed window | Count per time window | Simple, edge-of-window bursts |

### Pagination Patterns
- **Offset-based**: `?page=3&limit=50` — Simple but slow on large datasets (OFFSET is expensive)
- **Cursor-based**: `?cursor=eyJ0...&limit=50` — Efficient, stable (encode last-seen ID in cursor)
- **Keyset-based**: `?after_id=123&limit=50` — Fast (index seek), but requires sortable ID

**Staff insight**: Always use cursor-based for production APIs. Offset breaks with concurrent inserts and degrades at scale.

---

## 5. Observability at Scale

### The Three Pillars + Modern Extensions

```
Metrics → Prometheus/Cloud Monitoring → Dashboards + Alerts
  (counters, gauges, histograms)

Logs → Loki/CloudWatch/ELK → Search + Analysis
  (structured JSON, correlation IDs)

Traces → Jaeger/Cloud Trace → Latency analysis + Dependencies
  (distributed tracing, span propagation)

[Modern] Profiles → Continuous profiling → Performance optimization
  (CPU, memory, allocation flamegraphs)

[Modern] Events → Business events → Product analytics
  (user actions, system events, audit)
```

### SLI/SLO/SLA Framework
```
SLI (Service Level Indicator):
  - Availability: successful_requests / total_requests
  - Latency: proportion of requests < threshold
  - Quality: proportion of correct responses (for AI: hallucination rate)

SLO (Service Level Objective):
  - "99.9% of requests complete successfully within 500ms"
  - Error budget: 0.1% failure allowed = 43 min downtime/month

SLA (Service Level Agreement):
  - Contractual commitment with penalties
  - Always set SLA weaker than SLO (buffer)
```

### RED Method (for request-driven services)
- **R**ate: Requests per second
- **E**rrors: Failed requests per second
- **D**uration: Distribution of request latencies

### USE Method (for resources: CPU, memory, disk, network)
- **U**tilization: % time resource is busy
- **S**aturation: Queue length (work waiting)
- **E**rrors: Error count

---

## 6. Security Architecture

### Zero Trust Model
- Never trust, always verify
- Every request authenticated + authorized (even internal)
- Least privilege access
- Assume breach

### Authentication Patterns
```
External users → OAuth 2.0 / OIDC → JWT tokens
Service-to-service → mTLS (mutual TLS) → Certificate-based identity
API consumers → API keys + OAuth → Scoped tokens with rotation
```

### Encryption
- **At rest**: AES-256, per-tenant keys (envelope encryption with KMS)
- **In transit**: TLS 1.3 everywhere (including internal traffic)
- **In use**: Consider for highly sensitive data (customer pricing)

### Supply Chain Specific Security
- **Data isolation**: Tenant A must never see Tenant B's supply chain data
- **Audit logging**: Who accessed what, when (SOX compliance for public companies)
- **IP allowlisting**: Restrict ERP integration endpoints
- **Secret rotation**: API keys to ERP systems must rotate without downtime

---

## 7. Cost Optimization

### Cloud Cost Architecture Decisions
| Decision | Impact |
|----------|--------|
| Reserved vs on-demand instances | 30-60% savings for predictable workloads |
| Spot/preemptible for batch ML training | 60-90% savings |
| Auto-scaling with proper metrics | Avoid over-provisioning |
| Data tiering (hot/warm/cold) | 80%+ storage savings |
| Right-sizing instances | 20-40% savings |

### LLM Cost Optimization
```
Strategy                         Savings
─────────────────────────────    ───────
Prompt caching (prefix caching)  40-60%
Semantic caching                 50-80% for repeated queries
Model routing (simple → cheap)   60-70%
Shorter prompts (concise)        30-40%
Batch API for non-real-time      50%
Fine-tuning (smaller model)      80-90% (upfront cost)
```

### Cost per Query Estimation
```
Simple query (cache hit):         $0.001
RAG query (retrieval + LLM):      $0.02-0.05
Complex analytical (SQL + LLM):   $0.05-0.15
Recommendation (optimization):    $0.10-0.50

At 10K queries/hour:
- Best case (80% cache): $0.004/avg = $40/hour = $960/day
- Worst case (no cache): $0.05/avg = $500/hour = $12K/day
```

---

## 8. Testing Distributed Systems

### Testing Pyramid at Scale
```
                 /\
                /  \  E2E Tests (few, expensive, slow)
               /    \   - Full system integration
              /──────\  - Staging environment
             /        \
            / Contract  \ Contract Tests (moderate)
           /   Tests     \  - API compatibility
          /───────────────\  - Schema evolution
         /                 \
        /  Integration Tests \ Integration (many)
       /                      \  - Component pairs
      /────────────────────────\  - Real DBs, mocked externals
     /                          \
    /        Unit Tests          \ Unit (very many, fast)
   /______________________________\  - Business logic
```

### Chaos Engineering (Netflix-style)
```
Principles:
1. Define "steady state" (normal metrics)
2. Hypothesize steady state continues under stress
3. Inject failures: kill pods, add latency, partition network
4. Observe: did the system maintain steady state?

Supply Chain Example:
- Kill the ERP integration service → system should degrade gracefully
- Add 5s latency to ML model → fallback to last-known prediction
- Partition between regions → each region operates independently
```

### Load Testing Strategy
```
Levels:
1. Smoke test: 1 user, basic flows (sanity)
2. Load test: Expected peak (e.g., 3x normal)
3. Stress test: Beyond peak, find breaking point
4. Soak test: Normal load for 24h (memory leaks, connection exhaustion)

Tools: k6, Locust, Gatling
Key: Always test with production-like data volumes
```

---

## 9. Migration Strategies

### Strangler Fig Pattern (ERP Migration)
```
Before:
Users → Old ERP (monolith)

During:
Users → Routing Layer → Old ERP (declining traffic)
                      → New AI System (increasing traffic)

After:
Users → New AI System
```

### Data Migration Patterns
- **Big bang**: Migrate all at once (downtime)
- **Trickle**: Dual-write, gradual cutover
- **Shadow**: Write to both, compare results, switch reads

### Zero-Downtime Database Migration
```
1. Create new schema (additive changes only)
2. Deploy code that writes to BOTH old and new
3. Backfill historical data to new schema
4. Verify consistency (shadow reads)
5. Switch reads to new schema
6. Stop writing to old schema
7. Clean up old schema
```

**Staff insight**: Always have a rollback plan. "What if step 4 shows inconsistencies?" → stop migration, fix, restart from step 3.


---


# Moonshot Thinking for X Interviews

## What is X's Culture?

X (the moonshot factory) isn't a typical tech company. They:
1. **Seek 10x improvements**, not 10% improvements
2. **Embrace failure** — "fail fast, learn faster"
3. **Work on hard problems** — if it's easy, someone else will do it
4. **Operate at the intersection of technology and the physical world**
5. **Use first-principles thinking** — question everything assumed as given

## How This Shows Up in Interviews

### The "Novel Problem" Question
X may give you a system design problem that has no standard solution. Unlike Google's "design YouTube" (well-documented), you might get:

- "Design a system to predict global food waste 2 weeks in advance"
- "Design a system to optimally route perishable goods across 50 countries with different regulations"
- "Design an AI system that learns a company's supply chain from scratch with minimal human input"

### How to Handle Novel Problems

#### Step 1: Don't Panic
The interviewer doesn't expect you to know the answer. They want to see your THINKING PROCESS.

#### Step 2: Decompose from First Principles
```
"Design a system to predict supply chain disruptions before they happen"

First principles:
- What causes disruptions? (supplier failure, weather, geopolitics, demand shock)
- What signals precede disruptions? (news, weather forecasts, financial data, port congestion)
- How far in advance can we realistically predict? (hours to weeks, depends on type)
- What actions can be taken with advance warning? (re-route, pre-order, buffer stock)

→ This naturally decomposes into:
  1. Signal collection (multi-source data ingestion)
  2. Risk modeling (probabilistic, per disruption type)
  3. Impact assessment (what breaks if X happens)
  4. Action recommendation (what to do about it)
```

#### Step 3: Think in Orders of Magnitude
```
Bad: "We'd monitor some news feeds"
Good: "There are approximately 10M news articles published daily, 
      500K shipping events, 10K weather events. We need to filter 
      this down to the ~100 signals/day that actually matter for 
      our customer's supply chain. That's a 100,000:1 signal-to-noise 
      ratio, which tells me we need a multi-stage filtering pipeline."
```

#### Step 4: Acknowledge Uncertainty Explicitly
```
"I'm not sure what the best approach for X is, but here are three 
hypotheses I'd want to test:
1. [approach A] — this assumes [assumption]
2. [approach B] — this would work if [condition]  
3. [approach C] — this is most conservative

I'd probably start with [B] because [reasoning], but instrument 
heavily so we can learn whether [assumption] holds."
```

---

## The "10x Thinking" Framework

### Ask: What Would Make This Problem Trivially Easy?

**Regular thinking**: "How do we forecast demand 5% more accurately?"
**10x thinking**: "What if every product had a digital twin that simulated its entire lifecycle?"

**Regular thinking**: "How do we integrate with our customer's SAP system?"
**10x thinking**: "What if AI could understand ANY enterprise system by observing its behavior, without manual integration?"

### The "Magic Wand" Exercise
If you had unlimited compute, data, and time:
1. What would the ideal system look like?
2. Work backwards to what's possible today
3. Identify the specific barriers (data, compute, physics, regulation)
4. Propose the path that gets closest to ideal within constraints

---

## Handling "I Don't Know" at Staff Level

### The Wrong Way
"I don't know how to do that."

### The Right Way
"I haven't worked with that specific system before, but here's how I'd approach it based on analogous problems I've solved..."

### The Staff Way
```
"I don't have direct experience with [X], but let me reason about it:

1. The core problem is [fundamental challenge]
2. This is analogous to [related problem I know well]
3. The key differences are [specific deltas]
4. I'd hypothesize that [approach] would work because [reasoning]
5. The risks I'd want to validate first are [specific risks]

If I were building this, my first week would be:
- Day 1-2: Talk to domain experts about [specific questions]
- Day 3-4: Prototype [specific component] to validate [assumption]
- Day 5: Decide on architecture based on learnings"
```

---

## System Design for Uncertain Requirements

### The Spiral Approach
When requirements are ambiguous (which they will be at X):

```
                 Learn
                 /    \
              Build   Measure
             /              \
          ┌─────────────────────┐
          │  Sprint 1: MVP      │  ← Validate core assumption
          │  Sprint 2: Scale    │  ← Validate scalability  
          │  Sprint 3: Refine   │  ← Optimize from real data
          │  Sprint 4: Expand   │  ← New use cases
          └─────────────────────┘
```

### Design for Learning, Not Just Production
```
V0 (Week 1): Wizard-of-Oz prototype
  - Human does the AI's job manually
  - Validate: Do users actually want this?
  - Cost: Near-zero

V1 (Month 1): Rule-based system
  - Hardcoded rules from domain experts
  - Validate: Can this problem be automated at all?
  - Learn: What are the edge cases?

V2 (Month 3): ML-powered
  - Train on data collected in V1
  - Validate: Does ML actually beat rules?
  - Learn: Where does it fail?

V3 (Month 6): Production-grade
  - Scale, reliability, monitoring
  - Now we know WHAT to build
```

**Staff insight**: The most expensive mistake is building the wrong thing perfectly. At X, the first goal is to "de-risk the riskiest assumption."

---

## Domain-Specific Moonshot Questions to Prepare For

### 1. "Design a Self-Healing Supply Chain"
A system that automatically detects disruptions and reroutes/reorders without human intervention.

Key challenges:
- Confidence thresholds for autonomous action
- Liability and audit trail
- Cascading failures (your fix causes someone else's disruption)
- Human override and trust-building

### 2. "Design an AI That Learns Your Supply Chain from Data Alone"
Given access to a company's ERP and historical data, automatically map their supply chain topology and operations.

Key challenges:
- Schema diversity (every SAP instance is customized differently)
- Implicit knowledge (tribal knowledge not in any system)
- Validation (how do you know the mapping is correct?)
- Incremental learning as new data flows in

### 3. "Design a System to Reduce Food Waste by 50%"
End-to-end system from farm to store shelf.

Key challenges:
- Multi-stakeholder coordination (farmers, distributors, retailers)
- Perishability constraints (time is a hard constraint)
- Data availability (many participants have no digital systems)
- Incentive alignment (who pays for the system? who benefits?)

### 4. "Design a Supply Chain Digital Twin"
A real-time simulation of an entire supply chain.

Key challenges:
- Data freshness (simulation diverges from reality quickly)
- Fidelity vs compute cost
- What-if simulations (parallel universes)
- Human behavior modeling (people aren't predictable)

---

## Interview Communication for Moonshot Problems

### Structure Your Response
```
"Let me break this down into three layers:

First, the CORE PROBLEM: [1-2 sentences on what we're really solving]

Second, the HARD PARTS: [list 3-4 genuine challenges]
  - [Challenge 1] — this is hard because [reason]
  - [Challenge 2] — no one has solved this because [reason]

Third, my APPROACH:
  - Start with [simplest thing that could work]
  - Validate by [specific experiment]
  - Scale by [specific mechanism]

The key bet I'm making is [explicit assumption].
If that's wrong, I'd pivot to [alternative approach]."
```

### Phrases That Signal Staff-Level Thinking
- "The riskiest assumption here is..."
- "If I could only validate one thing first, it would be..."
- "This is analogous to [well-solved problem], except for [key difference]..."
- "The constraint that makes this hard is [specific constraint]..."
- "I'd instrument [X] to learn [Y] before committing to [Z]..."
- "The failure mode I'm most worried about is..."
- "At our target scale, the bottleneck shifts from [A] to [B]..."

---

## Cultural Signals X Looks For

### Intellectual Humility
- Admit what you don't know
- Ask smart questions
- Be willing to change your mind with new information

### Bias Toward Action
- Propose experiments, not just analysis
- "I'd build [quick prototype] to learn [specific thing]"
- Show momentum toward solutions

### Systems Thinking
- Consider second-order effects
- Think about incentives and human behavior
- Consider the full lifecycle, not just the happy path

### Collaborative Leadership
- "I'd want to talk to [domain expert] about [specific question]"
- "This decision needs input from [stakeholder] because [reason]"
- Mentor mindset: "I'd structure this so a junior engineer could own [component]"


---


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


---


# Mock Interview Questions (Self-Practice)

## Instructions
- Set a timer for 45 minutes per question
- Talk out loud (or write on paper/whiteboard)
- Use the framework from `01-framework-and-methodology.md`
- After each attempt, review the key points below

---

## TIER 1: Most Likely for This Role (Practice These First)

### Q1: Design an Enterprise Knowledge Base with AI Q&A
**Prompt**: "Our customers have thousands of documents — SOPs, contracts, supplier reviews, logistics manuals. Design a system that lets them search and ask questions in natural language."

**Key points to hit**:
- Multi-tenant data isolation (each enterprise customer's docs are private)
- Ingestion pipeline (PDF/Word/email → chunking → embedding → vector store)
- Hybrid retrieval (vector + keyword + metadata filtering)
- Answer quality (citations, confidence scores, hallucination prevention)
- Scale: 100 enterprises × 100K docs each × concurrent users
- Feedback loop: thumbs up/down → improve retrieval quality over time
- Access control: not everyone in a company should see all documents
- Freshness: when a document is updated, how quickly does the search reflect it?

**Staff-level differentiators**:
- Propose evaluation framework (offline + online metrics)
- Discuss cold start (new customer with zero feedback data)
- Consider the human workflow: what does the user do AFTER getting an answer?
- Multi-modal: some documents have tables, images, charts

---

### Q2: Design a Demand Forecasting System
**Prompt**: "Design a system that predicts demand for 100K SKUs across 500 locations, with daily granularity, looking 30 days ahead."

**Key points to hit**:
- Feature engineering (lag features, calendar, promotions, weather, economic indicators)
- Model architecture (ensemble: statistical + ML + foundation model)
- Hierarchical forecasting (total → category → SKU; reconciliation)
- Batch training pipeline (nightly retrain on latest data)
- Serving: pre-computed batch predictions + on-demand refresh capability
- Monitoring: forecast accuracy tracking, drift detection, alert on degradation
- Explainability: why is demand predicted to spike next Tuesday?
- Cold start: new product with no history

**Staff-level differentiators**:
- Discuss the organizational impact: who consumes forecasts? How do they use them?
- Feedback loop: did actual demand match prediction? Automated retraining triggers.
- Business metrics, not just ML metrics (inventory cost saved, stockout reduction)
- Probabilistic forecasts (confidence intervals) vs point estimates

---

### Q3: Design a Real-Time Supply Chain Event Processing System
**Prompt**: "We receive millions of events daily from warehouses, carriers, and stores. Design a system that processes these events in real-time to detect anomalies and trigger actions."

**Key points to hit**:
- Ingestion: heterogeneous sources (APIs, webhooks, file drops, IoT)
- Event schema: standardization, validation, enrichment
- Stream processing: Kafka + Flink for windowed aggregations
- Anomaly detection: statistical (Z-score) + ML (isolation forest)
- Action engine: rule-based + ML-based decision making
- Alerting: routing to right person, dedup, escalation
- Scale: 1M events/minute peak, <1 minute detection latency
- Dead letter queue: handle malformed/unprocessable events

**Staff-level differentiators**:
- Event ordering guarantees (what if events arrive out of order?)
- Exactly-once processing semantics (checkpointing strategy)
- Back-pressure handling (what if processing can't keep up?)
- Schema evolution (how to add new event types without breaking consumers)

---

### Q4: Design an AI-Powered ERP Integration Layer
**Prompt**: "Our customers use different ERP systems (SAP, Oracle, NetSuite). Design a system that connects to any ERP and normalizes the data for our AI platform."

**Key points to hit**:
- Anti-corruption layer (translate ERP-specific schemas to canonical model)
- Connector architecture (plugin-based: one connector per ERP type)
- Data extraction: CDC vs API polling vs file-based (trade-offs)
- Schema mapping: AI-assisted mapping from ERP fields to canonical model
- Incremental sync (only new/changed data, not full dumps)
- Error handling: what if ERP is down? Data quality issues?
- Security: credentials management, VPN/private link to on-prem ERPs
- Multi-tenant: shared infrastructure, isolated data

**Staff-level differentiators**:
- "AI-assisted onboarding": use LLM to auto-suggest field mappings
- Discuss the long tail: major ERPs have known schemas, but every instance is customized
- Data validation: detect and flag when ERP data doesn't match expected patterns
- Propose a "connector SDK" so new ERP integrations can be built by the team rapidly

---

## TIER 2: Classic Google-Style (Still Very Possible)

### Q5: Design a Notification System
**Prompt**: "Design a notification system that sends 1B notifications per day across push, email, and SMS."

**Key points to hit**:
- API for notification creation (template, recipient, channel, priority)
- Rate limiting per user (don't spam)
- Priority queues (critical alerts before marketing)
- Channel routing (user preferences, fallback strategy)
- Template engine (personalization, localization)
- Delivery tracking (sent, delivered, read, failed)
- Scale: 1B/day = 12K QPS, but bursty (100K QPS peak)
- Retry strategy per channel (email retry longer than push)

---

### Q6: Design a Rate Limiter
**Prompt**: "Design a distributed rate limiter for an API gateway handling 100K QPS."

**Key points to hit**:
- Algorithms: token bucket (API rate limiting), sliding window (more accurate)
- Distributed: Redis-based with Lua scripts for atomicity
- Consistency: slightly over-counting OK vs under-counting (fail open vs closed)
- Multi-dimension: per-user, per-IP, per-endpoint, global
- Response: HTTP 429 with Retry-After header
- Race conditions: what if two nodes both check simultaneously?
- Performance: Must add <1ms latency (Redis is fine)

---

### Q7: Design a Distributed Task Scheduler
**Prompt**: "Design a system that executes millions of scheduled tasks per day with at-least-once execution guarantee."

**Key points to hit**:
- Task persistence (durable storage: PostgreSQL or DynamoDB)
- Time-based partitioning (upcoming tasks hot, old tasks cold)
- Worker fleet (pull-based with lease/heartbeat mechanism)
- Exactly-once vs at-least-once (usually at-least-once with idempotent tasks)
- Failure handling: missed execution window, retry, dead-letter
- Scale: partition by scheduled time + hash for distribution
- Cron-style recurring tasks: expand to individual executions
- Visibility: task status API, execution history

---

### Q8: Design a Web Crawler
**Prompt**: "Design a web crawler that indexes 1B pages with freshness requirements."

**Key points to hit**:
- URL frontier (priority queue: important pages crawled more frequently)
- Politeness: respect robots.txt, rate limit per domain
- Deduplication: URL normalization, content fingerprinting (simhash)
- Architecture: distributed workers, shared URL frontier
- Storage: raw pages → S3, metadata → distributed DB, links → graph DB
- Freshness: re-crawl frequency based on page change rate
- Scale: 1B pages, 10KB avg = 10 PB. Incremental, not all at once.

---

## TIER 3: Moonshot/Novel Problems (X-Specific)

### Q9: Design a System to Predict Supply Chain Disruptions
**Prompt**: "Design a system that predicts supply chain disruptions 2 weeks before they happen."

**Approach hints**:
- Multi-source signal collection (news, weather, port data, financial markets, social media)
- NLP pipeline for news analysis (entity extraction, sentiment, event detection)
- Risk scoring model (per supplier, per route, per region)
- Graph-based propagation (if tier-2 supplier disrupted, which tier-1 affected?)
- Uncertainty quantification (probability + severity + confidence)
- Action recommendations (alternative suppliers, buffer stock, re-routing)
- Feedback loop: did predicted disruption materialize?

---

### Q10: Design a System That Automatically Maps a Company's Supply Chain
**Prompt**: "Given access to a company's ERP database and documents, design a system that automatically discovers and maps their entire supply chain network."

**Approach hints**:
- Data sources: purchase orders, invoices, shipping records, contracts
- Entity extraction: suppliers, warehouses, products, routes
- Relationship inference: who supplies what to whom, through what path
- Graph construction: nodes (entities) + edges (flows)
- Validation: present discovered topology to domain experts for confirmation
- Incremental: learn from corrections, update as new data flows in
- Visualization: interactive graph UI for exploration
- Anomaly: detect unexpected paths (potential fraud, non-compliance)

---

### Q11: Design a "What-If" Simulation Engine
**Prompt**: "Design a system where supply chain managers can ask 'what if supplier X shuts down?' or 'what if demand doubles next month?' and see the impact across their entire supply chain."

**Approach hints**:
- Digital twin: model of current supply chain state
- Simulation engine: discrete event simulation or system dynamics
- Scenario API: parameterized perturbations (remove node, scale demand, add lead time)
- Compute: simulations may take seconds to minutes → async with caching
- Visualization: before/after comparison, cascade visualization
- Recommendations: "here's how to mitigate this scenario"
- Scale: run 1000 Monte Carlo scenarios for probabilistic impact

---

## Self-Evaluation Rubric

After each practice session, score yourself:

| Criterion | 1 (Weak) | 3 (Adequate) | 5 (Strong) |
|-----------|----------|--------------|-------------|
| Requirements | Didn't clarify | Asked basics | Proactively scoped, quantified |
| Structure | Rambling | Organized | Clear phases, signposted transitions |
| Depth | Surface-level | Reasonable detail | Deep on hard problems, quantified trade-offs |
| Breadth | Missing components | All major components | Cross-cutting concerns (security, ops, cost) |
| Trade-offs | None discussed | Mentioned | Quantified, compared alternatives with reasoning |
| Communication | Confusing | Clear | Collaborative, invited discussion, structured |
| Staff signals | Junior-level | Senior-level | Drove direction, identified risks, thought org-wide |
| AI/ML knowledge | Generic | Solid fundamentals | Production-grade depth, aware of pitfalls |

**Target**: Score 4+ on all criteria. Focus your remaining study time on any criterion scoring <3.

---

## Practice Schedule (Fits Your 7-Day Plan)

| Day | Practice |
|-----|----------|
| Day 1 | Read frameworks, sketch Q6 (rate limiter) as warm-up |
| Day 2 | Full 45-min: Q5 (notification system) |
| Day 3 | Full 45-min: Q1 (enterprise knowledge base) — CRITICAL |
| Day 4 | Full 45-min: Q2 (demand forecasting) — CRITICAL |
| Day 5 | Full 45-min: Q3 (event processing) + Q4 (ERP integration) |
| Day 6 | Full 45-min: Q9 or Q10 (moonshot problem) |
| Day 7 | Review weak areas, re-do one problem cleanly |


---


# Behavioral & Staff-Level Signals for X Interviews

## Why This Matters

At Staff level, ~40% of the hiring signal comes from HOW you communicate and lead, not just WHAT you design. X especially values people who can navigate ambiguity with clarity.

---

## The "Googliness" + X Culture Signals

### Google-Inherited Values
- **Intellectual humility**: "I was wrong about X because Y"
- **Data-driven decisions**: Default to measurement over opinion
- **Bias toward action**: Ship and iterate, don't just plan
- **Collaboration**: "How would you work with [team X] on this?"

### X-Specific Values
- **Moonshot thinking**: Think 10x, not 10%
- **Kill early**: Willingness to shut down approaches that aren't working
- **Learn fast**: Rapid experimentation over long planning cycles
- **T-shaped people**: Deep expertise + broad curiosity
- **Ethical AI**: Responsible deployment, especially in enterprise contexts

---

## Staff-Level Behavioral Questions (Prepare Stories)

### Leadership Without Authority
**"Tell me about a time you influenced a technical decision across teams."**

Structure (STAR):
- Situation: [Context — what was the cross-cutting problem?]
- Task: [Your role — why did YOU take this on vs someone else?]
- Action: [Specific steps — meetings, docs, prototypes, data you gathered]
- Result: [Outcome — quantified impact, what changed]

Good answer includes:
- Understanding different teams' incentives
- Building alignment through shared goals, not mandate
- Data and prototypes over PowerPoints
- Explicitly stating what you'd do differently

---

### Navigating Ambiguity
**"Tell me about a time you started a project with unclear requirements."**

Good answer includes:
- How you SCOPED the ambiguity (what's known vs unknown)
- How you made progress WITHOUT full clarity
- Explicit assumptions you stated
- How you validated assumptions early
- When you sought more clarity vs when you just decided

---

### Technical Disagreement
**"Tell me about a time you disagreed with a senior engineer's technical approach."**

Good answer includes:
- Respectful framing (not "they were wrong")
- Your reasoning (data, principles, prior experience)
- How you expressed disagreement constructively
- The outcome (even if you were overruled — what did you learn?)
- What you'd do differently

---

### Mentorship & Growing Others
**"How have you helped other engineers grow?"**

At Staff level, this is expected:
- Code reviews that teach, not just approve
- Designing systems so others can own components
- Sponsoring (not just mentoring) — putting junior engineers forward for opportunities
- Creating documentation/runbooks that scale your knowledge

---

## How to Discuss Your Supply Chain / AI Experience

### If You Have Direct Experience
Frame it around IMPACT:
```
"At [Company], I designed [system] that [quantified outcome].
The key challenge was [technical/org challenge].
The approach I took was [what + why].
In hindsight, I'd change [specific learning]."
```

### If You Don't Have Direct Supply Chain Experience
Frame it through ANALOGIES:
```
"I haven't worked directly in supply chain, but the problems are 
structurally similar to [your experience]. For example:
- Demand forecasting ≈ [similar prediction problem you've solved]
- ERP integration ≈ [legacy system migration you've done]  
- Real-time optimization ≈ [similar optimization you've built]

The patterns transfer: [specific pattern], [specific pattern].
The domain-specific aspects I'd need to learn are: [specific gaps]."
```

---

## Questions to Ask Your Interviewer

### About the Role
- "What does the first 90 days look like for this role?"
- "What's the biggest technical challenge the team is facing right now?"
- "How do you balance moonshot ambition with shipping incrementally?"

### About the Team/Culture
- "How does X decide when to kill a project vs persist?"
- "What does the feedback loop look like — how quickly do you know if something is working?"
- "How autonomous are engineers in choosing technical approaches?"

### About the Product/Mission
- "What's the hardest part of working with enterprise customers as a moonshot?"
- "Where are you on the journey from prototype to product-market fit?"
- "What's the most surprising thing you've learned from early customers?"

### Signals These Questions Send
- You're thinking about IMPACT, not just code
- You're thinking about team dynamics, not just technical fit
- You're genuinely curious about the mission
- You're evaluating fit bidirectionally

---

## The "Why X?" Answer

Prepare a crisp answer for "Why do you want to work at X?"

### Template
```
"Three reasons:

1. PROBLEM: Supply chains are one of the largest systems in the world —
   $10T+ globally — and still largely run on spreadsheets and tribal knowledge.
   The AI opportunity here is massive, and X is one of the few places with 
   the ambition AND resources to actually tackle it at global scale.

2. APPROACH: I'm drawn to X's model of combining cutting-edge AI with 
   real-world enterprise problems. Most AI companies are either pure research 
   or pure incremental SaaS. X is doing something harder and more impactful.

3. PERSONAL: I thrive in ambiguous, early-stage environments where I can 
   shape direction. I've done [relevant experience] and I'm energized by 
   the idea of building something from the ground up that could transform 
   how the world moves goods."
```

---

## Day-Of Interview Tips

### Logistics
- Join 5 minutes early (test audio/video)
- Have paper/whiteboard ready (even for virtual — you may want to sketch)
- Water nearby
- Close everything else (no distractions)

### Mindset
- You're having a technical conversation with a colleague, not being interrogated
- The interviewer WANTS you to succeed
- Silence is OK — take 30 seconds to think before responding
- It's OK to say "Let me think about that for a moment"

### Recovery Moves
If you realize you went down a wrong path:
```
"Actually, let me step back. I was heading toward [approach], but 
I realize [problem with it]. A better approach would be [new direction] 
because [reasoning]."
```
This is a POSITIVE signal — it shows self-awareness and intellectual honesty.

### Energy Management
- System design round is mentally exhausting
- If it's back-to-back rounds, use the 5-min breaks to physically move
- Eat protein before interviews (stable energy)
- The afternoon slump is real — practice at the time your interview will be


---


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
- Multi-tenant SaaS, 50-100 enterprise customers
- Each has 10-50 concurrent users
- Mix of structured (ERP data) and unstructured (contracts, SOPs)
- Daily data refresh minimum, real-time for critical metrics
- Must cite sources, high accuracy, recommend only (no auto-actions)
- SOC 2 required, US data residency
- 10-15 seconds acceptable for complex queries

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
- 100 customers × 50 users × ~5 queries/hour = ~25K queries/hour = ~7 QPS avg
- Peak: 10x = 70 QPS  
- Latency: <5s for simple lookups, <15s for complex analysis
- Availability: 99.9% (44 min downtime/month)
- Data isolation: Tenant A must NEVER see Tenant B's data

COST ESTIMATE:
- 25K queries/hour × $0.03/query (LLM) = $750/hour = $18K/day
- With 60% cache hit rate: $7.2K/day
- This is acceptable for enterprise SaaS ($50K+/year per customer)

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


---


# Mock Interview 2: Real-Time Demand Forecasting System

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "Design a demand forecasting system for a large retailer. They have 100,000 SKUs across 500 store locations. They need daily forecasts looking 30 days ahead, updated in near-real-time as new sales data comes in. The system should handle promotions, seasonality, and external factors like weather."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About the Business:**
1. "What industry — grocery (perishable, daily replenishment), fashion (seasonal, long lead times), or electronics (high-value, intermittent demand)?"
2. "What decisions does the forecast drive — inventory replenishment, staffing, manufacturing?"
3. "What's the current forecasting method and its accuracy? What's the improvement target?"
4. "What's the cost of over-forecasting vs under-forecasting? Are they symmetric?"

**About Data:**
5. "What POS data do we have — transaction-level or daily aggregates? How far back?"
6. "Do we have promotion calendars in advance? How far ahead are they planned?"
7. "What external data is available — weather, economic indicators, local events?"
8. "Are there SKU lifecycle events — new product launches, discontinuations?"

**About Technical Requirements:**
9. "What's 'near-real-time'? Are we talking minutes, hours, or same-day?"
10. "Do we need probabilistic forecasts (confidence intervals) or just point estimates?"
11. "What's the downstream integration — does this feed directly into an auto-ordering system?"
12. "Hierarchical forecasting needed? (total company → category → subcategory → SKU → SKU-location)"

**About Constraints:**
13. "Cloud environment — GCP, AWS, hybrid?"
14. "Do we have a data science team to maintain models, or should this be mostly automated?"
15. "Any real-time data access constraints from POS systems?"

### Interviewer's Likely Answers:
- Grocery retailer (perishable goods, daily replenishment)
- 3 years of daily POS data, transaction-level
- Promotions planned 2-4 weeks ahead
- Under-forecasting costs 3x more than over-forecasting (stockouts are expensive)
- "Near-real-time" = within 2 hours of demand signal change
- Need probabilistic forecasts for safety stock calculations
- GCP environment
- Small data science team (3-4 people) — automation important

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me frame the problem quantitatively:

SCALE:
- 100K SKUs × 500 locations = 50 MILLION time series to forecast
- Each needs 30-day-ahead daily predictions = 50M × 30 = 1.5B forecast values/day
- POS data: ~10M transactions/day (500 stores × 20K transactions/store)

LATENCY:
- Batch forecast: Nightly run, ready by 6 AM for morning replenishment orders
- Incremental update: Within 2 hours of demand signal change

ACCURACY TARGET:
- Current baseline: ~65% MAPE at SKU-store-day level (typical for basic methods)
- Target: <50% MAPE (each 1% improvement ≈ significant inventory cost reduction)
- Note: grocery demand at SKU-day level is inherently noisy; 
  aggregated accuracy will be much higher

COMPUTE BUDGET:
- Training: 50M models × ~1 sec/model = ~14,000 compute-hours/week
  (Or: 1 global model trained on all series = much cheaper)
- Inference: 1.5B predictions in <6 hours = ~70K predictions/second
  (This is achievable with batch inference on GPU cluster)

Does this framing align with what you're thinking?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DATA INGESTION LAYER                               │
│                                                                       │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ POS Stream │  │ Promotion    │  │ External     │  │ Product   │ │
│  │ (Kafka)    │  │ Calendar     │  │ Data APIs    │  │ Master    │ │
│  │ 10M txn/day│  │ (scheduled)  │  │ (weather,    │  │ Data      │ │
│  │            │  │              │  │  events)     │  │ (ERP)     │ │
│  └─────┬──────┘  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
└────────┼────────────────┼───────────────────┼───────────────┼───────┘
         │                │                   │               │
         ▼                ▼                   ▼               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FEATURE ENGINEERING LAYER                          │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ STREAM PROCESSING (Flink)                                      │  │
│  │ - Real-time aggregation: sales velocity, trend detection       │  │
│  │ - Anomaly flags: unusual demand spikes                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ BATCH PROCESSING (Spark)                                       │  │
│  │ - Lag features: demand_t-1, demand_t-7, demand_t-28            │  │
│  │ - Rolling stats: mean_7d, std_7d, trend_28d                    │  │
│  │ - Calendar: day_of_week, month, holiday, payday                │  │
│  │ - Price: current_price, discount_pct, price_vs_competitors     │  │
│  │ - Promotion: promo_active, promo_type, days_since_promo_start  │  │
│  │ - Weather: temp, precipitation, forecast_next_7d               │  │
│  │ - Product: category, shelf_life, substitutes, cannibalization  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ FEATURE STORE (Feast on BigQuery + Redis)                      │  │
│  │ - Offline: BigQuery (training, backfill)                       │  │
│  │ - Online: Redis (serving, <10ms lookup)                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MODEL LAYER                                        │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │            MODEL ENSEMBLE                                     │    │
│  │                                                               │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │    │
│  │  │ Statistical │  │ ML (Gradient │  │ Foundation Model   │  │    │
│  │  │ (Prophet,   │  │  Boosted)    │  │ (TimesFM /        │  │    │
│  │  │  ETS)       │  │ LightGBM     │  │  Chronos)         │  │    │
│  │  │             │  │              │  │                    │  │    │
│  │  │ Good for:   │  │ Good for:    │  │ Good for:          │  │    │
│  │  │ - Trend     │  │ - Feature-   │  │ - Zero-shot        │  │    │
│  │  │ - Season    │  │   rich series│  │ - Cold start        │  │    │
│  │  │ - Holidays  │  │ - Promotions │  │ - Cross-category    │  │    │
│  │  │ - Simple    │  │ - Complex    │  │   patterns          │  │    │
│  │  │   patterns  │  │   interactions│  │ - Transfer learning │  │    │
│  │  └──────┬──────┘  └──────┬───────┘  └─────────┬──────────┘  │    │
│  │         │                │                     │              │    │
│  │         ▼                ▼                     ▼              │    │
│  │  ┌─────────────────────────────────────────────────────┐     │    │
│  │  │ META-LEARNER (Stacking)                              │     │    │
│  │  │ Learns optimal weights per SKU-store based on        │     │    │
│  │  │ historical accuracy of each base model               │     │    │
│  │  │ Output: Quantile forecasts (p10, p50, p90)           │     │    │
│  │  └─────────────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ MODEL REGISTRY (MLflow / Vertex AI Model Registry)           │    │
│  │ - Versioned models, A/B test configurations                  │    │
│  │ - Rollback capability                                        │    │
│  │ - Performance tracking per model version                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SERVING & CONSUMPTION LAYER                        │
│                                                                       │
│  ┌───────────────┐  ┌────────────────────┐  ┌───────────────────┐  │
│  │ Forecast API  │  │ Replenishment       │  │ Monitoring &      │  │
│  │ (gRPC/REST)   │  │ System Integration  │  │ Dashboards        │  │
│  │ <50ms latency │  │ (auto-reorder       │  │ (accuracy,        │  │
│  │ (pre-computed)│  │  triggers)          │  │  drift, alerts)   │  │
│  └───────────────┘  └────────────────────┘  └───────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions to State:

```
"A few important architectural decisions:

1. WHY AN ENSEMBLE: No single model wins across all 50M series. 
   Statistical models handle simple/stable series well. ML models 
   capture complex feature interactions (promotions). Foundation 
   models handle cold-start. The meta-learner learns which to trust per series.

2. WHY BATCH + INCREMENTAL: Base forecasts run nightly (compute-efficient,
   full feature set). Intraday updates are INCREMENTAL — adjust the 
   base forecast based on today's demand deviation, don't rerun everything.

3. WHY QUANTILE FORECASTS: The business needs to set safety stock.
   Safety stock = f(uncertainty). Point estimates don't capture uncertainty.
   We output p10/p50/p90 so downstream systems can make risk-appropriate 
   inventory decisions.

4. WHY FEATURE STORE: Training-serving consistency is the #1 cause of 
   ML production failures. Same feature definitions for offline training 
   and online serving eliminates this risk."
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Hierarchical Forecasting & Reconciliation

**Why this matters**: Individual SKU-store forecasts are noisy. Aggregated forecasts (category, region) are more accurate. They must be consistent.

```
┌─────────────────────────────────────────────────────┐
│          FORECAST HIERARCHY                          │
│                                                      │
│  Level 0: Total company demand                       │
│      ↓                                               │
│  Level 1: Category (Dairy, Bakery, Produce, ...)     │
│      ↓                                               │
│  Level 2: Subcategory (Milk, Cheese, Yogurt, ...)    │
│      ↓                                               │
│  Level 3: SKU (Brand X Whole Milk 1L)                │
│      ↓                                               │
│  Level 4: SKU-Location (SKU at Store #42)            │
│                                                      │
│  PROBLEM: Sum of Level 4 forecasts ≠ Level 0 forecast│
│  (they disagree because different models/granularity) │
└─────────────────────────────────────────────────────┘

RECONCILIATION APPROACHES:

1. TOP-DOWN: Forecast at top, disaggregate proportionally
   Pro: Smooth, stable. Con: Loses local patterns.

2. BOTTOM-UP: Forecast at SKU-location, sum up
   Pro: Captures local detail. Con: Noisy, sums may be unrealistic.

3. OPTIMAL RECONCILIATION (MinT - Minimum Trace):
   - Forecast at ALL levels independently
   - Use optimal combination weights to reconcile
   - Minimize total forecast variance across all levels
   - Proven to improve accuracy at every level

   Mathematically:
   ŷ_reconciled = S × (S'ΣS)^(-1) × S' × ŷ_base
   
   Where S is the summing matrix, Σ is forecast error covariance

MY CHOICE: Optimal reconciliation (MinT approach)
- 5-15% accuracy improvement over bottom-up at no extra model complexity
- Guarantees consistency across all levels
- Additional compute cost: one matrix operation per forecast cycle (minutes)
```

---

### Deep Dive 2: Handling Promotions

**Interviewer might ask**: "Promotions cause huge demand spikes. How do you handle them?"

```
"Promotions are the hardest part of demand forecasting. Here's my approach:

THE CHALLENGE:
- Promotion effect can be 2-10x normal demand
- Effects vary by: promo type, discount depth, display location, 
  timing, competitive activity
- Cannibalization: promo on SKU-A steals from SKU-B
- Pantry loading: spike during promo, dip after (pull-forward demand)
- Halo effect: promo on one item lifts the category

FEATURE ENGINEERING FOR PROMOTIONS:
┌───────────────────────────────────────────────┐
│ Promo Features (computed from promotion calendar)│
├───────────────────────────────────────────────┤
│ promo_active: boolean                          │
│ promo_type: {display, price_cut, BOGO, flyer}  │
│ discount_depth: 0.0 - 1.0                      │
│ days_into_promo: 1, 2, 3...                    │
│ days_until_promo_end: countdown                 │
│ competing_promos: count of same-category promos │
│ last_promo_same_sku: days since                 │
│ last_promo_lift: historical lift for this SKU   │
│ promo_display_type: {endcap, inline, feature}   │
└───────────────────────────────────────────────┘

MODEL ARCHITECTURE FOR PROMOTIONS:

Option A: Include promo features in the ensemble (simpler)
  - LightGBM naturally handles feature interactions
  - Works well for SKUs with promo history

Option B: Separate promo uplift model (more sophisticated)
  ┌─────────────────────────────────────────┐
  │ Final Forecast = Base Forecast × Lift   │
  │                                          │
  │ Base: "What would demand be WITHOUT      │
  │        promotion?" (trained on non-promo │
  │        periods only)                     │
  │                                          │
  │ Lift: "Given this promo configuration,   │
  │        what's the multiplier?"           │
  │        (trained on promo periods)        │
  │                                          │
  │ Pro: Interpretable, handles new promos   │
  │ Con: Assumes base and lift are independent│
  └─────────────────────────────────────────┘

MY CHOICE: Option B for mature SKUs, Option A for new/infrequent promos.
The lift model is more interpretable for business users who need to plan.

HANDLING CANNIBALIZATION:
- Cross-SKU features: when SKU-A is on promo, include that as a feature 
  for SKU-B's forecast (same subcategory)
- Train a cannibalization matrix: % of uplift that comes from substitutes
  (This can be learned from historical promo patterns)

POST-PROMO DIP:
- Feature: days_since_last_promo × last_promo_lift
- Model learns that large promos create demand dips in subsequent days
- Include in base forecast after promo ends
"
```

---

### Deep Dive 3: Near-Real-Time Forecast Updates

**Interviewer might ask**: "You said 2-hour updates. How does that work without retraining?"

```
"The key insight: we separate MODEL TRAINING from FORECAST ADJUSTMENT.

┌─────────────────────────────────────────────────────────────┐
│              TWO-SPEED FORECASTING                            │
│                                                              │
│  BATCH (Nightly, 1:00-5:00 AM):                             │
│  ─────────────────────────────                               │
│  - Retrain ensemble on latest complete day                   │
│  - Full feature computation (all lagged features)            │
│  - Generate base forecasts for all 50M series × 30 days     │
│  - Store in Forecast DB (Bigtable, keyed by SKU-store-date)  │
│  - Reconcile hierarchy                                       │
│  - Compute accuracy metrics vs actuals                       │
│                                                              │
│  INCREMENTAL (Every 2 hours, 8 AM - 10 PM):                 │
│  ────────────────────────────────────────────                │
│  - Ingest last 2 hours of POS data                           │
│  - Compare actual demand TODAY vs base forecast for today     │
│  - If deviation > threshold:                                 │
│    1. Classify: signal (real trend) vs noise (random)        │
│    2. If signal: adjust remaining forecast days              │
│       Adjustment = f(deviation_today, historical patterns)   │
│    3. Propagate to affected hierarchy levels                 │
│    4. Notify downstream (replenishment system)               │
│                                                              │
│  SIGNAL vs NOISE DETECTION:                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Cumulative deviation: track running sum of                │ │
│  │ (actual - forecast) throughout the day.                   │ │
│  │                                                           │ │
│  │ If |cumulative_deviation| > 2σ (based on historical      │ │
│  │ intraday variability for this SKU):                       │ │
│  │   → Signal! Something changed.                            │ │
│  │   → Investigate: new promo started? Weather event?        │ │
│  │                  Competitor stockout? Viral social post?   │ │
│  │                                                           │ │
│  │ If within normal band:                                    │ │
│  │   → Noise. Don't adjust. (Avoid over-reacting)            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ADJUSTMENT ALGORITHM:                                       │
│  Simple exponential smoothing on the forecast error:          │
│  adjusted_forecast[t+h] = base_forecast[t+h]                 │
│                          + α^h × (actual[t] - base[t])       │
│  where α ∈ (0, 1) decays the adjustment over horizon         │
│  (today's surprise matters less for 30 days from now)        │
│                                                              │
└─────────────────────────────────────────────────────────────┘

WHY NOT JUST RETRAIN EVERY 2 HOURS?
- 50M series × model training = massive compute (14K GPU-hours)
- Retraining on intraday data risks overfitting to noise
- Incremental adjustment is O(N) simple arithmetic, not O(N × training)
- Cost: $0.01 for adjustment vs $10K for full retrain
- Only retrain when LONG-TERM pattern changes (drift detection triggers)
"
```

---

## PHASE 5: Monitoring & Wrap-Up (5 minutes)

### Model Monitoring:
```
"Critical monitoring for forecasting systems:

ACCURACY TRACKING:
- Daily MAPE/WMAPE per category, store, overall
- Bias tracking: are we systematically over/under forecasting?
- Accuracy stratified by: promotion vs non-promotion
                          new products vs established
                          high volume vs long tail

DRIFT DETECTION:
- Feature drift: distributions of input features changing?
  (e.g., average basket size declining → economic downturn?)
- Concept drift: relationship between features and demand changing?
  (e.g., weather sensitivity increasing due to climate change?)
- Action: alert when drift detected → trigger investigation and potential retrain

BUSINESS METRICS (most important):
- Stockout rate: direct impact of under-forecasting
- Days of inventory: direct impact of over-forecasting  
- Waste rate (perishables): forecasts too high → product expires
- These are what the business ACTUALLY cares about, not MAPE

ALERTING:
- Accuracy degrades >5% week-over-week → auto-investigate
- Specific SKU/store accuracy <threshold → flag for review
- External event detected (hurricane, strike) → auto-adjust forecasts
  for affected regions
"
```

### Failure Modes:
```
"Key failure scenarios and mitigations:

1. POS data delayed:
   → Fallback to last-known base forecast (no incremental update)
   → Alert data engineering team
   → Forecast is stale but not catastrophically wrong

2. Model training fails:
   → Serve yesterday's model (still valid)
   → Models degrade slowly, not suddenly
   → Alert ML team, auto-retry

3. Demand shock (pandemic, viral event):
   → Anomaly detector catches it
   → Switch to 'regime change' mode: overweight recent data
   → Human-in-the-loop for extreme events
   → Graceful degradation: 'forecast confidence: LOW'

4. New product (cold start):
   → Use foundation model (zero-shot from similar products)
   → Fallback: category average × store traffic
   → Rapidly adjust as first days of data arrive
"
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Scale estimation | 50M series, quantified compute, realistic latency targets | ☐ |
| Model architecture | Ensemble rationale, not just "use deep learning" | ☐ |
| Feature engineering | Specific, relevant features with business justification | ☐ |
| Hierarchical | Reconciliation approach, consistency guarantee | ☐ |
| Promotions | Detailed handling, cannibalization, post-promo dip | ☐ |
| Real-time updates | Signal vs noise, efficient adjustment, not naive retrain | ☐ |
| Monitoring | Business metrics alongside ML metrics | ☐ |
| Cold start | Foundation model, category average, rapid adaptation | ☐ |
| Staff signals | Phased delivery, team implications, business impact | ☐ |


---


# Mock Interview 3: Real-Time Supply Chain Event Processing & Anomaly Detection

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "Design a system that ingests millions of events per day from warehouses, carriers, suppliers, and stores. It should detect anomalies in real-time, predict issues before they become critical, and automatically trigger corrective actions. Think of it as an immune system for the supply chain."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Event Sources:**
1. "What types of events? I'm thinking: inventory movements, shipment status updates, order events, IoT sensor data (temperature, humidity). Anything else?"
2. "How heterogeneous are the sources? Are they all APIs, or do some push via webhooks, SFTP files, or IoT protocols (MQTT)?"
3. "What's the expected event volume? I'll estimate, but want to confirm: millions per day or per hour?"
4. "Is the data clean, or do we need significant validation/enrichment?"

**About Anomaly Detection:**
5. "What counts as an 'anomaly'? Simple threshold violations? Statistical deviations? Predicted future issues?"
6. "What's the acceptable false positive rate? Too many false alarms → alert fatigue."
7. "Detection latency target — is 'within 5 minutes of event' acceptable, or do we need sub-second?"
8. "Should the system LEARN new anomaly patterns over time, or are patterns pre-defined?"

**About Actions:**
9. "What 'corrective actions' can the system take? Alerts only? Or autonomous actions like rerouting shipments, adjusting orders?"
10. "Who needs to be notified? What channels — Slack, email, SMS, pager?"
11. "Escalation policy — if no acknowledgment in 15 minutes, escalate?"
12. "What's the blast radius of a wrong action? Can we auto-revert?"

**About Scale & Environment:**
13. "How many customers (if multi-tenant)? Or is this for one large enterprise?"
14. "GCP/AWS/on-prem?"
15. "Existing infrastructure we should integrate with?"

### Expected Answers:
- Single large enterprise initially, expanding to multi-tenant later
- 5M events/day currently, growing to 50M within 18 months
- Mix of sources: APIs, webhooks, batch files, IoT sensors
- Detection within 5 minutes acceptable
- Initial actions: alert only. Phase 2: auto-actions for low-risk decisions
- GCP environment
- Some events arrive out of order (shipment updates can be delayed)

---

## PHASE 2: Requirements Summary & Math (3 minutes)

```
"Let me quantify this:

SCALE:
- Current: 5M events/day = ~58 events/sec avg, ~300/sec peak
- Target (18 mo): 50M events/day = 580 events/sec avg, 3000/sec peak
- This is well within Kafka's capacity (millions/sec)

EVENT SIZE:
- Avg event: 500 bytes (JSON with metadata)
- Daily storage: 50M × 500B = 25 GB/day = 9 TB/year
- Retention: 90 days hot + archive

ANOMALY DETECTION:
- 50M events/day, assume 0.1% are anomalous = 50K potential anomalies
- With 95% precision target: 2,500 false positives/day ≈ 100/hour
- That's still too many for humans → need intelligent grouping and severity

LATENCY BUDGET:
- Event ingestion: <1 sec
- Processing + enrichment: <1 min
- Anomaly detection: <3 min
- Alert delivery: <1 min
- Total: <5 min end-to-end ✓

Does this match your expectations?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌────────────────────────────────────────────────────────────────────────┐
│                       DATA SOURCE LAYER                                  │
│                                                                          │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌────────┐  ┌────────┐ │
│  │Warehouse │  │ Carrier   │  │ Supplier  │  │  IoT   │  │  ERP   │ │
│  │  WMS     │  │ Tracking  │  │ Portals   │  │Sensors │  │Updates │ │
│  │(webhook) │  │ (API poll)│  │(SFTP/API) │  │ (MQTT) │  │ (CDC)  │ │
│  └────┬─────┘  └────┬──────┘  └────┬──────┘  └───┬────┘  └───┬────┘ │
└───────┼──────────────┼──────────────┼─────────────┼────────────┼──────┘
        │              │              │             │            │
        ▼              ▼              ▼             ▼            ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    INGESTION LAYER                                       │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  CONNECTOR FLEET (Kafka Connect / Custom Connectors)           │    │
│  │  - Webhook receiver (HTTP endpoint for push sources)            │    │
│  │  - API poller (scheduled fetch for pull sources)                │    │
│  │  - SFTP watcher (file-based sources)                            │    │
│  │  - MQTT bridge (IoT → Kafka)                                    │    │
│  │  - CDC connector (Debezium for ERP databases)                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                              │                                           │
│                              ▼                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  KAFKA CLUSTER                                                  │    │
│  │  Topics:                                                        │    │
│  │  - raw-events (partitioned by source_type)                      │    │
│  │  - validated-events (partitioned by entity_id)                  │    │
│  │  - anomalies (partitioned by severity)                          │    │
│  │  - actions (partitioned by action_type)                         │    │
│  │  Config: 50 partitions, 3x replication, 7-day retention         │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                STREAM PROCESSING LAYER (Apache Flink)                    │
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │
│  │ VALIDATION &      │  │ ENRICHMENT       │  │ STATEFUL             │ │
│  │ NORMALIZATION     │  │                  │  │ AGGREGATION          │ │
│  │                   │  │ - Add entity     │  │                      │ │
│  │ - Schema check    │  │   metadata       │  │ - Tumbling: 5min     │ │
│  │ - Deduplication   │  │ - Geo-resolve    │  │   counts per entity  │ │
│  │ - Canonical format│  │ - Category map   │  │ - Sliding: 1hr       │ │
│  │ - Missing fields  │  │ - Historical     │  │   moving averages    │ │
│  │ - Dead letter     │  │   context lookup │  │ - Session: per-      │ │
│  │   (malformed)     │  │                  │  │   shipment lifecycle │ │
│  └──────────┬────────┘  └────────┬─────────┘  └──────────┬───────────┘ │
│             │                    │                        │              │
│             ▼                    ▼                        ▼              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                ANOMALY DETECTION ENGINE                           │   │
│  │                                                                   │   │
│  │  Layer 1: Rule-Based (immediate, deterministic)                   │   │
│  │  ┌─────────────────────────────────────────────────────────┐     │   │
│  │  │ - Temperature > threshold (perishables at risk)          │     │   │
│  │  │ - Shipment not scanned in 24h (stuck in transit)         │     │   │
│  │  │ - Inventory < safety stock (imminent stockout)           │     │   │
│  │  │ - Order rejected by supplier (supply disruption)         │     │   │
│  │  └─────────────────────────────────────────────────────────┘     │   │
│  │                                                                   │   │
│  │  Layer 2: Statistical (near-real-time, adaptive)                  │   │
│  │  ┌─────────────────────────────────────────────────────────┐     │   │
│  │  │ - Z-score: event rate deviation from expected pattern    │     │   │
│  │  │ - CUSUM: cumulative sum for gradual trend shifts         │     │   │
│  │  │ - Seasonal decomposition: is this deviation > normal     │     │   │
│  │  │   for this time/day/season?                              │     │   │
│  │  └─────────────────────────────────────────────────────────┘     │   │
│  │                                                                   │   │
│  │  Layer 3: ML-Based (complex patterns, learned)                    │   │
│  │  ┌─────────────────────────────────────────────────────────┐     │   │
│  │  │ - Isolation Forest: multivariate outlier detection        │     │   │
│  │  │ - Sequence model: unusual event sequences                │     │   │
│  │  │ - Predictive: forecast next state, alert if unlikely     │     │   │
│  │  │ - Graph-based: anomalous patterns in supply network      │     │   │
│  │  └─────────────────────────────────────────────────────────┘     │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    ACTION & NOTIFICATION LAYER                           │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ DECISION ENGINE                                                    │  │
│  │                                                                    │  │
│  │ Anomaly → Severity Assessment → Correlation → Action Selection    │  │
│  │              │                       │              │              │  │
│  │        (critical/                (group        (alert /            │  │
│  │         high/medium/              related       auto-action /      │  │
│  │         low)                      anomalies)    suppress)          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌──────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Alert Router │  │ Auto-Actions   │  │ Escalation Manager       │  │
│  │ (Slack/Email/│  │ (reroute,      │  │ (if no ack in 15 min,   │  │
│  │  SMS/PagerD) │  │  reorder,      │  │  escalate to manager)   │  │
│  │              │  │  adjust ETA)   │  │                          │  │
│  └──────────────┘  └────────────────┘  └──────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    STORAGE & ANALYTICS LAYER                             │
│                                                                          │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Hot Store  │  │ Warm Store   │  │ Cold Store   │  │ Analytics  │ │
│  │ (Bigtable) │  │ (BigQuery)   │  │ (GCS)        │  │ (Looker)   │ │
│  │ 7 days     │  │ 90 days      │  │ Years        │  │ Dashboards │ │
│  │ Low latency│  │ Queryable    │  │ Archive      │  │            │ │
│  └────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Anomaly Detection — Multi-Layer Approach

**Why multi-layer?**
```
"No single detection method works for all anomaly types. I use three layers,
each with different strengths:

┌─────────────────────────────────────────────────────────────────┐
│ DETECTION ACCURACY vs SPEED TRADE-OFF                            │
│                                                                   │
│  Layer 1: Rules          │  Speed: Immediate (milliseconds)      │
│  - Fixed thresholds       │  Accuracy: High precision, low recall  │
│  - Known failure modes    │  Coverage: Only pre-defined patterns   │
│  - Example: temp > 40°C   │  Cost: Near-zero compute              │
│                           │                                        │
│  Layer 2: Statistical     │  Speed: Seconds (needs window data)    │
│  - Z-score, CUSUM         │  Accuracy: Good, some false positives  │
│  - Adaptive baselines     │  Coverage: Any metric with history     │
│  - Seasonal adjustment    │  Cost: Low compute (simple math)       │
│                           │                                        │
│  Layer 3: ML Models       │  Speed: Minutes (batch window needed)  │
│  - Isolation Forest       │  Accuracy: High recall, catches subtle │
│  - Sequence models        │  Coverage: Multivariate, complex       │
│  - Predictive anomalies   │  Cost: Moderate compute (model infer)  │
└─────────────────────────────────────────────────────────────────┘

DETAILED EXAMPLE — Detecting a Shipment Delay Before It Happens:

Event stream for shipment SHP-12345:
  t0: departed_warehouse (Chicago, expected transit: 3 days)
  t1: +6h: scanned_hub (Indianapolis) ← on schedule
  t2: +18h: scanned_hub (Columbus) ← on schedule
  t3: +30h: NO SCAN ← expected: scanned_hub (Pittsburgh)

Layer 1 (Rule): "No scan in 12h when expected" → flag
Layer 2 (Statistical): "For this route, 98% of shipments scan 
         Pittsburgh by hour 28. At hour 30, p-value < 0.02" → flag
Layer 3 (ML): "Sequence model predicts 85% probability of ≥1 day delay
         based on: (1) no Pittsburgh scan, (2) weather alert in PA,
         (3) carrier historically delayed on this route in storms"

Each layer adds information:
- Rule tells us: something is wrong
- Stats tell us: how unusual this is
- ML tells us: probable root cause and expected delay magnitude

COMBINING DETECTIONS:
- If ANY layer fires → create candidate anomaly
- Severity = f(layers_triggered, confidence, business_impact)
- If all 3 layers agree → HIGH severity
- If only rules → MEDIUM (might be transient)
- If only ML → LOW (investigate, might be noise)
"
```

---

### Deep Dive 2: Event Ordering & Exactly-Once Processing

**Interviewer might ask**: "How do you handle out-of-order events?"

```
"Out-of-order events are inevitable in distributed systems. Here's how I handle them:

CAUSES OF OUT-OF-ORDER:
- Network delays (carrier API batches updates every 30 min)
- Clock skew between source systems
- Retry mechanisms delivering late duplicates
- Batch file processing (entire day's events arrive at once)

APPROACH: Event-Time Processing with Watermarks

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  WALL CLOCK:  |-------|-------|-------|-------|                    │
│               10:00   10:05   10:10   10:15                       │
│                                                                   │
│  EVENTS ARRIVING (event_time → arrival_time):                     │
│    Event A (09:55) arrives at 10:01  ← on time                   │
│    Event B (09:57) arrives at 10:03  ← on time                   │
│    Event C (09:52) arrives at 10:08  ← LATE (6 min delay)        │
│    Event D (09:58) arrives at 10:02  ← on time                   │
│                                                                   │
│  WATERMARK: "I believe I've received all events up to time T"     │
│    At 10:05, watermark = 09:58 (conservative estimate)            │
│    At 10:10, watermark = 10:03 (advancing)                        │
│                                                                   │
│  WINDOWED AGGREGATION (5-min tumbling window: 09:55-10:00):       │
│    - Events A, B, D are in this window                            │
│    - Window closes when watermark > 10:00                         │
│    - Event C arrives AFTER window closed → LATE EVENT             │
│                                                                   │
│  LATE EVENT HANDLING:                                              │
│    Option 1: Drop (simple, may miss anomalies)                    │
│    Option 2: Allowed lateness (recompute window, update results)  │
│    Option 3: Side output (process separately, flag as late)       │
│                                                                   │
│  MY CHOICE: Allowed lateness of 1 hour + side output beyond       │
│    - Most late events arrive within minutes (normal network)      │
│    - Carrier batches can be up to 30-60 min late                  │
│    - Beyond 1 hour → side output for manual review                │
│    - Anomaly detection results are UPDATED when late events arrive│
└─────────────────────────────────────────────────────────────────┘

DEDUPLICATION:
- Every event has a unique event_id (UUID from source or generated)
- Flink maintains a Bloom filter of seen event_ids (last 24h)
- If event_id already seen → drop (idempotent processing)
- Bloom filter false positive rate: 0.1% (acceptable, slight over-counting)

EXACTLY-ONCE SEMANTICS:
- Flink checkpoints state to distributed storage (every 30s)
- On failure: restart from last checkpoint, replay from Kafka offset
- Kafka consumer offsets committed with checkpoint (atomic)
- Sinks (BigQuery, alerts): idempotent writes (upsert, dedup key)

Trade-off discussion:
- Exactly-once adds ~10% latency overhead (checkpoint barriers)
- For anomaly detection, at-least-once would also work 
  (processing same event twice = same anomaly detected twice = dedup at alert layer)
- I'd use exactly-once for aggregation COUNTS (must be accurate)
  and at-least-once for anomaly detection (idempotent downstream)
"
```

---

### Deep Dive 3: Alert Fatigue & Intelligent Notification

**Interviewer might ask**: "50K potential anomalies per day is way too many for humans. How do you make this useful?"

```
"Alert fatigue is the #1 failure mode of monitoring systems. My approach:

┌─────────────────────────────────────────────────────────────────┐
│            ANOMALY → ACTIONABLE ALERT PIPELINE                   │
│                                                                   │
│  Raw anomalies (50K/day)                                         │
│       │                                                           │
│       ▼                                                           │
│  STEP 1: DEDUPLICATION & GROUPING (→ 5K groups)                  │
│  ────────────────────────────────────────────                    │
│  - Same entity, same anomaly type, within 30 min → one group    │
│  - Related entities (shipment + its line items) → one group      │
│  - Same root cause (carrier delay + all affected shipments) →    │
│    one group                                                     │
│       │                                                           │
│       ▼                                                           │
│  STEP 2: SEVERITY SCORING (→ prioritized list)                   │
│  ────────────────────────────────────────────                    │
│  Score = f(business_impact, confidence, urgency)                  │
│                                                                   │
│  business_impact: estimated $ at risk                             │
│    - Shipment delay on $1M order > $100 order                    │
│    - Perishable goods > durable goods                            │
│    - Customer with SLA penalties > standard                      │
│                                                                   │
│  confidence: how sure are we this is real?                        │
│    - All 3 detection layers agree: 0.95                          │
│    - Only ML model: 0.6                                          │
│    - Only threshold rule on noisy metric: 0.4                    │
│                                                                   │
│  urgency: how soon must action be taken?                          │
│    - Perishable goods spoiling in 2 hours: CRITICAL              │
│    - Shipment delay affecting delivery tomorrow: HIGH            │
│    - Trend suggesting stockout in 2 weeks: MEDIUM                │
│       │                                                           │
│       ▼                                                           │
│  STEP 3: SUPPRESSION & CORRELATION (→ 500 alerts/day)            │
│  ──────────────────────────────────────────────────              │
│  - Known issues: suppress if root cause already acknowledged      │
│  - Maintenance windows: suppress expected anomalies               │
│  - Flapping: if anomaly resolves within 5 min, don't alert       │
│  - Cascade detection: if 20 shipments delayed because one         │
│    carrier is down → ONE alert about carrier, not 20 alerts      │
│       │                                                           │
│       ▼                                                           │
│  STEP 4: ROUTING & DELIVERY                                      │
│  ────────────────────────────                                    │
│  Route based on:                                                  │
│  - Severity: CRITICAL → PagerDuty + SMS                          │
│              HIGH → Slack + Email                                 │
│              MEDIUM → Daily digest                                │
│              LOW → Dashboard only                                 │
│  - Ownership: route to person responsible for this entity         │
│  - Schedule: respect working hours (non-critical → next morning)  │
│  - Preferences: some users want all alerts, some want critical only│
│       │                                                           │
│       ▼                                                           │
│  STEP 5: FEEDBACK LOOP                                           │
│  ─────────────────────                                           │
│  - User marks alert: "useful" / "not useful" / "too late"        │
│  - Feed back into severity model (learn what matters)             │
│  - Track: alert → action taken → outcome                         │
│  - Monthly: review suppressed alerts for missed critical ones     │
└─────────────────────────────────────────────────────────────────┘

NUMBERS:
- 50K raw anomalies → 5K groups → 500 alerts → 50 CRITICAL
- A supply chain manager sees ~10 alerts/day (actionable, relevant)
- CRITICAL pages are <5/day (only truly urgent)

THE KEY METRIC: Alert Precision
- % of alerts where user took action = measure of usefulness
- Target: >70% of alerts should result in user action
- If precision drops → model is producing noise → investigate
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

### System Evolution:
```
"Phase 1 (Month 1-3): Rule-based detection + alerting
  - Ship fast with known failure modes
  - Learn: what events exist? What do users care about?

Phase 2 (Month 3-6): Add statistical detection + smart routing
  - Adaptive baselines per entity
  - Severity scoring and grouping
  - Feedback loop from user responses

Phase 3 (Month 6-12): ML detection + autonomous actions
  - Train on accumulated labeled data (user feedback = labels)
  - Auto-actions for low-risk, high-confidence situations
  - Predictive capabilities (foresee issues before they happen)

Phase 4 (12+ months): Self-healing supply chain
  - System proposes AND executes corrective actions
  - Human approves via exception (inverse control)
  - Continuous learning from outcomes"
```

### Key Trade-offs Summary:
```
"The three hardest trade-offs in this system:

1. SENSITIVITY vs SPECIFICITY:
   - More sensitive → catch more real issues, but more false alarms
   - My choice: err on the side of sensitivity for CRITICAL, 
     specificity for LOW severity (different thresholds per tier)

2. SPEED vs ACCURACY:
   - Faster detection → less data to confirm, more uncertainty
   - My choice: three-tier speed (immediate rules, seconds stats, minutes ML)
   - Alert includes confidence level so user knows how certain we are

3. AUTOMATION vs CONTROL:
   - More automation → faster response, less human oversight
   - My choice: start manual, earn trust, gradually automate
   - Always: human can override any automated action
   - Never: automate actions above $ threshold without approval"
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Ingestion diversity | Multiple source types, connectors, schema normalization | ☐ |
| Stream processing | Flink, windowing, watermarks, exactly-once semantics | ☐ |
| Detection depth | Multi-layer (rules + stats + ML), specific algorithms | ☐ |
| Out-of-order handling | Watermarks, allowed lateness, dedup strategy | ☐ |
| Alert fatigue | Grouping, severity scoring, suppression, routing | ☐ |
| Actionability | Clear escalation, feedback loop, learning from outcomes | ☐ |
| Scale reasoning | 50M events → 50K anomalies → 500 alerts → 50 critical | ☐ |
| Evolution | Phased from manual to autonomous, trust-building | ☐ |


---


# Mock Interview 4: Global Logistics Route Optimization

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "Design a system that optimizes shipping routes and logistics decisions for a global supply chain network. Think multi-modal (truck, rail, ocean, air), across dozens of countries, with constraints like customs, perishability, cost, and time. The system should recommend optimal routes and adapt when disruptions occur."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Scope:**
1. "Are we optimizing for a single company's logistics, or building a platform for multiple companies?"
2. "What modalities: just truck? Or multi-modal (truck → port → ocean → port → truck)?"
3. "Scale: how many shipments per day? How many possible routes?"
4. "What's the optimization objective: minimize cost? Minimize time? Balance of both?"

**About Constraints:**
5. "Hard constraints I should know about: customs regulations, hazmat, temperature control, weight limits, driver hours-of-service?"
6. "Do we need to handle dynamic re-routing? E.g., port closure mid-shipment?"
7. "SLA commitments: guaranteed delivery dates that can't be violated?"
8. "Consolidation: can we combine multiple shipments on the same vehicle?"

**About Data:**
9. "What data do we have about routes: historical transit times, carrier performance, real-time traffic/weather?"
10. "How often do route costs change? (Fuel prices, carrier rates, spot market)"
11. "Do we have real-time visibility of vehicle positions?"

**About Users:**
12. "Who uses this: logistics planners? Or is it fully automated?"
13. "Do they need to see alternatives (route A vs B vs C) or just the best recommendation?"
14. "How far in advance are shipments planned vs on-demand?"

### Expected Answers:
- Multi-modal global logistics for a single large enterprise (expandable later)
- 10,000 shipments/day, routes can span 5+ countries
- Optimize for cost with delivery time constraints (SLA)
- Must handle disruptions with real-time re-routing
- Logistics planners review and approve recommendations
- Mix of planned (80%) and urgent (20%) shipments

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me frame the problem:

SCALE:
- 10K shipments/day needing route optimization
- Each route has 10-50 possible options (mode × carrier × path combinations)
- Route network: ~1000 nodes (warehouses, ports, hubs, DCs), ~10K edges
- Re-routing: ~500 shipments/day affected by disruptions

COMPUTE:
- Batch optimization (nightly for next-day planned shipments): 
  8K shipments × graph search = must complete in <2 hours
- Real-time routing (urgent + re-routes):
  ~100 requests/hour × <5 seconds response time
- This is a combinatorial optimization problem that's NP-hard in general,
  but solvable with heuristics at this scale

COST FACTORS:
- Transport cost: $0.50-$5.00/km (varies by mode)
- Time cost: $X/hour of inventory in transit (opportunity cost)
- Risk cost: probability of disruption × impact
- Penalty cost: SLA violation = $Y per day late

DATA FRESHNESS:
- Carrier rates: update daily
- Real-time: traffic, weather, port congestion (update every 15 min)
- Disruptions: immediate (event-driven)

Am I framing this correctly?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DATA & CONTEXT LAYER                               │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Route Network│  │ Real-Time    │  │ Carrier &    │  │ Disruption │ │
│  │ Graph        │  │ Conditions   │  │ Rate Data    │  │ Feed       │ │
│  │ (nodes,edges,│  │ (traffic,    │  │ (contracts,  │  │ (weather,  │ │
│  │  distances,  │  │  weather,    │  │  spot market,│  │  port close│ │
│  │  modes)      │  │  congestion) │  │  schedules)  │  │  strikes)  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
└─────────┼──────────────────┼──────────────────┼───────────────┼─────────┘
          │                  │                  │               │
          ▼                  ▼                  ▼               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      OPTIMIZATION ENGINE                                  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ ROUTE GRAPH ENGINE                                                │    │
│  │                                                                   │    │
│  │  Weighted multi-modal graph where:                                │    │
│  │  - Nodes = locations (warehouses, ports, hubs, customers)         │    │
│  │  - Edges = transport legs (mode, carrier, capacity, schedule)     │    │
│  │  - Edge weights = f(cost, time, risk, CO2) — DYNAMIC              │    │
│  │                                                                   │    │
│  │  Example edge: Chicago_WH → LA_Port                               │    │
│  │    Mode: Rail | Cost: $2,400 | Time: 72h | Reliability: 95%      │    │
│  │    Mode: Truck | Cost: $4,200 | Time: 30h | Reliability: 99%     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ SOLVER LAYER                                                      │    │
│  │                                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │    │
│  │  │ Level 1: Single Shipment Router                          │     │    │
│  │  │ - Modified Dijkstra with constraints (time windows, mode │     │    │
│  │  │   compatibility, customs requirements)                   │     │    │
│  │  │ - Returns: Top K routes with cost/time/risk trade-offs   │     │    │
│  │  │ - Latency: <500ms per shipment                           │     │    │
│  │  └─────────────────────────────────────────────────────────┘     │    │
│  │                                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │    │
│  │  │ Level 2: Fleet Optimizer (batch)                         │     │    │
│  │  │ - Vehicle Routing Problem (VRP) variant                  │     │    │
│  │  │ - Consolidation: combine compatible shipments            │     │    │
│  │  │ - Capacity: respect vehicle/container limits             │     │    │
│  │  │ - Algorithm: Constraint programming (OR-Tools) +         │     │    │
│  │  │   metaheuristics (Large Neighborhood Search)             │     │    │
│  │  │ - Latency: minutes to hours (nightly batch)              │     │    │
│  │  └─────────────────────────────────────────────────────────┘     │    │
│  │                                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │    │
│  │  │ Level 3: ML-Enhanced Predictions                         │     │    │
│  │  │ - Transit time prediction (actual vs scheduled)          │     │    │
│  │  │ - Disruption probability per route                       │     │    │
│  │  │ - Cost prediction (spot market rates)                    │     │    │
│  │  │ - Used to adjust edge weights in the graph               │     │    │
│  │  └─────────────────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ CONSTRAINT ENGINE                                                 │    │
│  │ - Hard constraints: customs (docs required), hazmat (prohibited   │    │
│  │   modes), temperature (reefer required), weight limits            │    │
│  │ - Soft constraints: preferred carriers, CO2 budget, cost caps     │    │
│  │ - SLA constraints: must arrive by date X (convert to time budget) │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   DISRUPTION RESPONSE LAYER                               │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ EVENT LISTENER (subscribes to disruption events from Kafka)       │    │
│  │                                                                   │    │
│  │ Disruption detected → Identify affected shipments →               │    │
│  │   Re-route using Level 1 solver (real-time, single shipment) →    │    │
│  │   Compare: new route vs wait-it-out →                             │    │
│  │   If savings > threshold: recommend re-route to planner           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     USER INTERFACE LAYER                                   │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Route Recommendation UI                                              │ │
│  │ ┌───────────────────────────────────────────────────────────────┐  │ │
│  │ │ Shipment: 40ft container, Shanghai → Chicago                   │  │ │
│  │ │ SLA: Deliver by June 25                                        │  │ │
│  │ │                                                                 │  │ │
│  │ │ Option A (Recommended):                                         │  │ │
│  │ │   Shanghai → Busan (truck, 1d) → LA (ocean, 14d) →             │  │ │
│  │ │   Chicago (rail, 3d) | $4,200 | 18 days | 94% on-time          │  │ │
│  │ │                                                                 │  │ │
│  │ │ Option B (Faster):                                              │  │ │
│  │ │   Shanghai → LA (ocean express, 11d) → Chicago (truck, 2d)     │  │ │
│  │ │   | $6,800 | 13 days | 97% on-time                             │  │ │
│  │ │                                                                 │  │ │
│  │ │ Option C (Cheapest):                                            │  │ │
│  │ │   Shanghai → Long Beach (ocean slow, 18d) → Chicago (rail, 4d) │  │ │
│  │ │   | $3,100 | 22 days | 88% on-time ⚠️ SLA RISK                │  │ │
│  │ └───────────────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: The Route Graph & Solver

```
"The core algorithmic challenge: find optimal paths through a multi-modal, 
time-constrained, capacity-constrained network.

GRAPH REPRESENTATION:
┌─────────────────────────────────────────────────────────────────┐
│ Node types:                                                      │
│   - Origin warehouse (OW): where shipment starts                │
│   - Hub/Port (H): transshipment points                          │
│   - Destination (D): final delivery location                    │
│                                                                  │
│ Edge types (each has temporal + capacity dimensions):             │
│   - Road: continuous availability, travel time varies by traffic │
│   - Rail: scheduled departures (e.g., daily at 6am, 2pm)        │
│   - Ocean: scheduled sailings (e.g., weekly Wed from Shanghai)   │
│   - Air: scheduled flights (multiple daily)                      │
│                                                                  │
│ TIME-EXPANDED GRAPH:                                             │
│ Because transport has SCHEDULES, I use a time-expanded graph:    │
│                                                                  │
│   Shanghai_Mon → (ocean, departs Mon) → LA_+14days               │
│   Shanghai_Wed → (ocean, departs Wed) → LA_+14days               │
│   Shanghai_Mon → (wait 2 days) → Shanghai_Wed                    │
│                                                                  │
│ This converts the scheduling problem into a graph problem:       │
│ "Find shortest path from Shanghai_now to Chicago_beforeSLA"      │
└─────────────────────────────────────────────────────────────────┘

SOLVER ALGORITHM (Single Shipment):

Modified A* with:
  - Edge weights: cost + α×time + β×risk + γ×CO2
  - Constraints as path validation (reject invalid paths)
  - Return top-K paths (not just optimal)

def find_routes(origin, destination, shipment, sla_deadline):
    # Build time-expanded subgraph (only relevant time window)
    graph = build_time_graph(origin, destination, 
                            start=now, end=sla_deadline)
    
    # Apply constraints as edge filters
    graph.filter_edges(
        mode_allowed=shipment.allowed_modes,
        max_weight=shipment.weight,
        requires_temp_control=shipment.perishable,
        customs_docs_available=shipment.customs_cleared_for
    )
    
    # Multi-objective: find Pareto-optimal set
    routes = pareto_optimal_search(
        graph, origin, destination,
        objectives=[cost, time, risk],
        constraint=arrival_before(sla_deadline),
        k=5  # top 5 diverse routes
    )
    
    return routes

PERFORMANCE:
- Graph size: ~1000 nodes × 168 hours (weekly) = 168K time-expanded nodes
- Edges: ~10K base × schedule frequency = ~100K time-expanded edges  
- A* on this graph: <100ms per search
- 10K shipments batch: parallelize across 50 workers = 200 seconds total

WHY NOT JUST USE GOOGLE MAPS ROUTING?
- Multi-modal (Google Maps doesn't do ocean + rail + customs)
- Schedule-aware (can't just book a ship like hailing an Uber)
- Consolidation (combine shipments to fill containers)
- Enterprise constraints (preferred carriers, contractual rates)
- Cost models (complex: fuel surcharge, currency, duties, insurance)
"
```

---

### Deep Dive 2: Dynamic Re-Routing on Disruption

```
"When a disruption occurs mid-transit, we need to rapidly re-route 
affected shipments. Here's the architecture:

DISRUPTION EVENT FLOW:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Disruption: "Port of LA closed due to labor strike"              │
│       │                                                           │
│       ▼                                                           │
│  STEP 1: IMPACT ASSESSMENT (<30 seconds)                         │
│  Query: "Which in-transit shipments pass through LA port?"        │
│  → Scan active shipments table (indexed by route nodes)          │
│  → Result: 47 shipments affected                                 │
│       │                                                           │
│       ▼                                                           │
│  STEP 2: TRIAGE (<1 minute)                                      │
│  For each affected shipment, classify urgency:                    │
│  - Already past LA: no impact ✓                                  │
│  - Arriving today: CRITICAL (reroute immediately)                │
│  - Arriving this week: HIGH (reroute within hours)               │
│  - Arriving next week: MEDIUM (monitor, may resolve)             │
│  → 12 CRITICAL, 20 HIGH, 15 MEDIUM                              │
│       │                                                           │
│       ▼                                                           │
│  STEP 3: ALTERNATIVE ROUTING (<5 minutes)                        │
│  For CRITICAL + HIGH (32 shipments):                              │
│  - Remove LA port from graph (mark edge as unavailable)          │
│  - Re-run solver for each shipment                               │
│  - Alternatives: Long Beach port, Oakland port, overland route   │
│       │                                                           │
│       ▼                                                           │
│  STEP 4: COST-BENEFIT ANALYSIS                                   │
│  For each shipment, compare:                                     │
│  - Re-route cost (new carrier, mode change, expediting)          │
│  - Wait cost (SLA penalty, inventory carrying, customer impact)  │
│  - Risk of waiting (how long might disruption last?)             │
│       │                                                           │
│       ▼                                                           │
│  STEP 5: RECOMMENDATION                                         │
│  Present to logistics planner:                                    │
│  "LA port strike — 32 shipments affected.                        │
│   Recommended: Re-route 12 CRITICAL via Long Beach (+$800 avg,   │
│   saves 3 days). Monitor 20 HIGH (strike likely <48h based on    │
│   historical pattern)."                                          │
│  [Approve All] [Review Each] [Override]                          │
│       │                                                           │
│       ▼                                                           │
│  STEP 6: EXECUTION                                               │
│  On approval:                                                     │
│  - Update shipment routes in system of record                    │
│  - Notify carriers (API calls to book new legs)                  │
│  - Update ETAs for downstream stakeholders                       │
│  - Log decision for audit trail                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

DISRUPTION DURATION PREDICTION (ML):
- Historical data: how long did similar disruptions last?
- Features: disruption type, location, season, severity indicators
- Model: survival analysis (predicts duration distribution)
- Use median predicted duration to decide: reroute vs wait

This prediction is CRITICAL for the wait-vs-reroute decision.
If strike will last 2 hours → wait. If 2 weeks → reroute everything.
"
```

---

### Deep Dive 3: Cost Modeling

```
"Accurate cost modeling is what makes route recommendations trustworthy.

COST COMPONENTS (per shipment leg):
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  TRANSPORT COST = base_rate + fuel_surcharge + accessorials       │
│    - base_rate: contracted (from rate table) or spot (API query)  │
│    - fuel_surcharge: fuel_index × rate_per_km × distance          │
│    - accessorials: pickup fee, delivery fee, detention, liftgate  │
│                                                                   │
│  TIME COST = inventory_value × carrying_rate × transit_days       │
│    - Opportunity cost of capital tied up in transit               │
│    - Higher for expensive goods (electronics > bulk materials)    │
│                                                                   │
│  RISK COST = P(disruption) × expected_impact                     │
│    - P(disruption): from ML model per route + conditions          │
│    - expected_impact: SLA penalty + customer churn + expedite cost│
│                                                                   │
│  HANDLING COST = terminal_fees + customs_brokerage + insurance    │
│    - Each transshipment point adds handling cost                  │
│    - More stops = more handling cost + more risk                  │
│                                                                   │
│  TOTAL COST = Σ(transport + time + risk + handling) over all legs │
│                                                                   │
│  CONSTRAINTS (hard):                                              │
│  - Must arrive by SLA deadline                                    │
│  - Must use reefer if perishable                                  │
│  - Must clear customs (documentation ready)                      │
│  - Cannot exceed weight/volume limits                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

RATE DATA MANAGEMENT:
- Contract rates: loaded from ERP, updated quarterly
- Spot rates: API calls to carrier marketplaces (DAT, Freightos)
- Rate cache: 1-hour TTL for spot rates (prices change frequently)
- ML prediction: forecast next-week spot rates for planning ahead

Example total cost calculation:
  Shanghai → Chicago (40ft container, electronics, $200K value):
  
  Leg 1: Shanghai → LA (ocean, 14 days)
    Transport: $2,800 (contract rate)
    Time: $200K × 8%/year × 14/365 = $614
    Risk: 5% × $5K penalty = $250
    Handling: $400 (port fees)
    Subtotal: $4,064
    
  Leg 2: LA → Chicago (rail, 3 days)
    Transport: $1,200
    Time: $200K × 8%/year × 3/365 = $132
    Risk: 2% × $5K = $100
    Handling: $200 (terminal)
    Subtotal: $1,632
    
  TOTAL: $5,696 | 17 days | 93% on-time probability
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

```
"Key trade-offs in this system:

1. OPTIMALITY vs SPEED:
   - True optimal (integer programming) might take hours for fleet-level
   - My choice: exact for individual routes (<500ms), heuristic for fleet
   - Good enough in real-time > perfect answer too late

2. RE-ROUTE FREQUENCY vs STABILITY:
   - Re-route too often → operational chaos, carrier penalties
   - Re-route too rarely → miss cost savings
   - My choice: re-evaluate on disruption events OR when potential savings 
     exceed threshold (>15% cost reduction)

3. CENTRALIZED vs DISTRIBUTED OPTIMIZATION:
   - Centralized: globally optimal, complex, single point of failure
   - Distributed: each region optimizes locally, may miss global optima
   - My choice: centralized solver with regional caches for real-time

Evolution path:
- V1: Single-shipment routing with alternatives (ship fast, learn)
- V2: Consolidation and fleet optimization (cost savings)
- V3: Predictive disruption avoidance (proactive re-routing)
- V4: Autonomous logistics (system executes without human approval)"
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Problem modeling | Time-expanded graph, multi-modal, scheduling | ☐ |
| Algorithm choice | A*/Dijkstra for single, VRP/OR-Tools for fleet | ☐ |
| Constraint handling | Hard vs soft constraints, customs, temp, weight | ☐ |
| Disruption response | Impact assessment → triage → re-route → execute | ☐ |
| Cost model | Multi-factor (transport + time + risk + handling) | ☐ |
| Real-time vs batch | Dual-speed: batch for planning, real-time for urgency | ☐ |
| User experience | Present options with trade-offs, not just "best" answer | ☐ |
| Staff signals | Phased evolution, stability vs optimality trade-off | ☐ |


---


# Mock Interview 5: Intelligent Document Processing Platform

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "Design a system that automatically processes supply chain documents — purchase orders, invoices, bills of lading, contracts, packing lists — extracts structured data, validates it against business rules, and feeds it into downstream systems. Handle any format: PDF, scanned images, emails, EDI. The system should be 99% accurate on critical fields and learn from corrections."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Volume & Types:**
1. "How many documents per day? What's the distribution across types?"
2. "Are these mostly digital-native PDFs or scanned paper (requires OCR)?"
3. "How many distinct document templates/layouts do we encounter?"
4. "Multi-language? Multilateral trade has docs in many languages."

**About Accuracy & Fields:**
5. "Which fields are 'critical' (99% accuracy required)? I'm guessing: amounts, dates, quantities, PO numbers?"
6. "What's the current process — fully manual data entry? What's the error rate today?"
7. "Is 99% accuracy per-field or per-document (all fields correct)?"
8. "How do we handle ambiguous or conflicting information within a document?"

**About Integration:**
9. "Where does extracted data go — ERP system, data warehouse, workflow engine?"
10. "Do we need to match documents to each other? (PO → Invoice → Receipt → Payment)"
11. "What validation rules exist? (e.g., invoice total = sum of line items)"
12. "Speed requirement: real-time (<1min) or batch (within hours)?"

**About Learning:**
13. "When humans correct AI mistakes, how should the system learn — immediately or in periodic retraining?"
14. "Do different customers have different document formats and rules?"
15. "How do we handle completely new document types we haven't seen before?"

### Expected Answers:
- 50K documents/day, growing to 200K
- 60% digital PDF, 30% scanned, 10% email/EDI
- ~500 distinct templates across all customers
- Critical fields: amounts (99%), dates (99%), PO/invoice numbers (99.5%), line items (95%)
- Currently: 40% manual data entry, 60% semi-automated (template matching)
- Multi-tenant: each customer has unique document formats
- Extracted data feeds into ERP and a matching/reconciliation engine
- Processing time: <5 minutes per document
- Learn from corrections within 24 hours

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me size this:

THROUGHPUT:
- 200K docs/day (target) = ~2.3 docs/sec sustained
- But bursty: most arrive 8am-6pm business hours = ~5.5 docs/sec
- Peak burst: 10x = 55 docs/sec (month-end invoice dumps)

PROCESSING BUDGET:
- 5 minutes per doc target
- Actual processing time per doc: 10-60 seconds (depending on complexity)
- Bottleneck is ML inference, not I/O

ACCURACY MATH:
- 99% accuracy on amounts means: 1 error per 100 documents
- At 200K docs/day = 2,000 errors/day needing human review
- Human review capacity: 20 reviewers × 100 docs/day = 2,000/day ✓
- Goal: drive auto-approval rate up (currently 60% → target 95%)

STORAGE:
- Avg document: 200KB (PDF) + 5KB (extracted JSON) + 1KB (metadata)
- Daily: 200K × 200KB = 40 GB/day for originals
- Keep originals 7 years (compliance) = ~100 TB
- Extracted data: negligible vs originals

COST:
- OCR/extraction per doc: $0.01-0.05 (depending on model)
- 200K × $0.03 = $6K/day = $180K/month
- Compare to: 40 manual data entry staff × $5K/month = $200K/month
- ROI is clear even before accuracy improvements

Good framing?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DOCUMENT INGESTION                                  │
│                                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────┐│
│  │  Email   │  │  SFTP    │  │  API     │  │  Scanner │  │  EDI  ││
│  │ (mailbox)│  │  (watch) │  │ (upload) │  │  (agent) │  │(parse)││
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬───┘│
│       └──────────────┴──────────────┴──────────────┴────────────┘    │
│                                     │                                 │
│                                     ▼                                 │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Document Queue (SQS/Pub-Sub)                                    │  │
│  │ - Priority: urgent POs > standard invoices > archival           │  │
│  │ - Retry: 3x with exponential backoff                            │  │
│  │ - DLQ: failed after retries → human review                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    PROCESSING PIPELINE                                 │
│                                                                        │
│  STAGE 1: PREPROCESSING                                               │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ - File type detection (PDF vs image vs email)                    │  │
│  │ - PDF rendering (convert to images for OCR if needed)            │  │
│  │ - Image enhancement (deskew, denoise, contrast for scanned docs) │  │
│  │ - Page splitting (multi-page docs → individual pages)            │  │
│  │ - Language detection                                              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  STAGE 2: DOCUMENT UNDERSTANDING (Multi-Model)                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │ MODEL A: Layout-Aware (LayoutLMv3 / DocFormer)           │    │  │
│  │  │ - Understands document structure (tables, headers, etc)  │    │  │
│  │  │ - Fine-tuned on supply chain document corpus              │    │  │
│  │  │ - Best for: structured forms, tables, standard layouts    │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │ MODEL B: Vision-Language Model (GPT-4V / Claude Vision)  │    │  │
│  │  │ - Understands arbitrary document formats                  │    │  │
│  │  │ - Zero-shot capability for new templates                  │    │  │
│  │  │ - Best for: irregular layouts, handwritten annotations    │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │ MODEL C: Template Matching (trained per document type)   │    │  │
│  │  │ - Highest accuracy for known templates                    │    │  │
│  │  │ - Fast, cheap inference                                   │    │  │
│  │  │ - Requires labeled examples per template                  │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                                                                  │  │
│  │  ROUTING LOGIC:                                                  │  │
│  │  - Known template (>80% confidence) → Model C (fast, accurate)  │  │
│  │  - Known doc type, unknown template → Model A (layout-aware)    │  │
│  │  - Unknown everything → Model B (VLM, zero-shot)                │  │
│  │  - Low confidence from any → escalate to human                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  STAGE 3: EXTRACTION & STRUCTURING                                    │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Output schema (enforced via structured output / function calling):│  │
│  │ {                                                                 │  │
│  │   "document_type": "invoice",                                     │  │
│  │   "header": {                                                     │  │
│  │     "invoice_number": {"value": "INV-2026-5678", "confidence": 0.99},│
│  │     "date": {"value": "2026-06-10", "confidence": 0.98},         │  │
│  │     "total_amount": {"value": 45230.00, "confidence": 0.97},     │  │
│  │     "currency": {"value": "USD", "confidence": 0.99},            │  │
│  │     "vendor": {"value": "Acme Supply Co", "confidence": 0.95}    │  │
│  │   },                                                              │  │
│  │   "line_items": [                                                 │  │
│  │     {"sku": "SKU-789", "qty": 100, "unit_price": 45.23,         │  │
│  │      "line_total": 4523.00, "confidence": 0.96}                  │  │
│  │   ],                                                              │  │
│  │   "metadata": {                                                   │  │
│  │     "page_count": 2,                                              │  │
│  │     "model_used": "template_match_v3",                            │  │
│  │     "processing_time_ms": 2300                                    │  │
│  │   }                                                               │  │
│  │ }                                                                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  STAGE 4: VALIDATION & ENRICHMENT                                     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Rule-based validation:                                           │  │
│  │ - Arithmetic: line_total = qty × unit_price (within $0.01)       │  │
│  │ - Sum: total_amount = Σ(line_totals) + tax + shipping            │  │
│  │ - Referential: PO number exists in our system                    │  │
│  │ - Format: dates are valid, amounts are positive                   │  │
│  │ - Business: vendor is in approved supplier list                   │  │
│  │                                                                   │  │
│  │ Cross-reference enrichment:                                       │  │
│  │ - Match vendor name to entity in master data (fuzzy matching)     │  │
│  │ - Match SKU descriptions to our product catalog                   │  │
│  │ - Look up PO to verify quantities and prices match                │  │
│  │                                                                   │  │
│  │ Confidence routing:                                                │  │
│  │ - ALL fields high confidence + validation passes → AUTO-APPROVE   │  │
│  │ - ANY critical field low confidence → HUMAN REVIEW                │  │
│  │ - Validation fails → FLAG + HUMAN REVIEW                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    HUMAN-IN-THE-LOOP LAYER                             │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Review Queue UI:                                                 │  │
│  │ - Show original document side-by-side with extracted fields      │  │
│  │ - Highlight low-confidence fields (red/yellow)                   │  │
│  │ - Allow inline correction (click field, type correct value)      │  │
│  │ - Keyboard shortcuts for speed (Tab, Enter to confirm)           │  │
│  │ - Smart ordering: highest business impact first                  │  │
│  │                                                                   │  │
│  │ Review effort metrics:                                            │  │
│  │ - Avg time per document review: 45 seconds                       │  │
│  │ - Auto-approval rate: target 95%                                  │  │
│  │ - Fields corrected per reviewed doc: avg 1.2                      │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    LEARNING & IMPROVEMENT LAYER                        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Active Learning Pipeline:                                         │  │
│  │                                                                   │  │
│  │ Human corrections → Correction Store → Nightly Retraining        │  │
│  │                         │                                         │  │
│  │                         ▼                                         │  │
│  │     Template Recognition: "Is this a new template?"              │  │
│  │       YES → Create template, fine-tune Model C                   │  │
│  │       NO → Add to existing template's training data              │  │
│  │                                                                   │  │
│  │ Feedback loop metrics:                                            │  │
│  │ - Track accuracy per template over time (should improve)         │  │
│  │ - Track correction patterns (systematic errors → model gap)      │  │
│  │ - New template detection (clustering unseen layouts)              │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Multi-Model Strategy & Routing

```
"Why three models instead of just using a VLM for everything?

COST-ACCURACY-SPEED MATRIX:
┌─────────────────┬──────────┬──────────┬──────────┬──────────────┐
│ Model           │ Accuracy │ Speed    │ Cost/doc │ When to Use   │
├─────────────────┼──────────┼──────────┼──────────┼──────────────┤
│ Template Match  │ 99.5%    │ 200ms    │ $0.001   │ Known layouts │
│ (fine-tuned)    │          │          │          │ (80% of docs) │
├─────────────────┼──────────┼──────────┼──────────┼──────────────┤
│ LayoutLM        │ 97%      │ 1-2s     │ $0.01    │ Known types,  │
│ (layout-aware)  │          │          │          │ new layouts   │
├─────────────────┼──────────┼──────────┼──────────┼──────────────┤
│ VLM (Claude/GPT)│ 95%      │ 5-15s    │ $0.03-   │ Unknown docs, │
│ (zero-shot)     │          │          │  $0.10   │ complex cases │
└─────────────────┴──────────┴──────────┴──────────┴──────────────┘

ROUTING DECISION TREE:
┌─────────────────────────────────────────────────────────────────┐
│  Document arrives                                                │
│       │                                                          │
│       ▼                                                          │
│  Template Fingerprint (hash of layout structure)                  │
│       │                                                          │
│       ├─ MATCH (confidence > 0.9) → Template Model (fast, cheap) │
│       │                                                          │
│       ├─ PARTIAL MATCH (0.6-0.9) → LayoutLM (moderate)           │
│       │                                                          │
│       └─ NO MATCH → VLM (expensive, but handles anything)        │
│                                                                  │
│  After extraction, regardless of model:                          │
│  - Validation rules run (arithmetic, referential)                │
│  - If validation fails + confidence was high → possible          │
│    model degradation → alert ML team                             │
└─────────────────────────────────────────────────────────────────┘

TEMPLATE FINGERPRINTING (how we quickly identify known templates):
- Extract: page dimensions, text block positions, logo hash, header pattern
- Create structural fingerprint (ignoring content, capturing layout)
- Compare to library of known fingerprints (cosine similarity)
- 200K docs/day × 200ms fingerprinting = 11 hours (easily parallelizable)

This tiered approach saves 80% of cost vs using VLM for everything,
while maintaining accuracy by routing hard cases to capable models.
"
```

---

### Deep Dive 2: Line Item Extraction (The Hardest Part)

```
"Line items in tables are the most challenging extraction task. Here's why and how:

CHALLENGES:
1. Table layouts vary wildly (column order, merged cells, multi-row items)
2. Some PDFs have invisible table structure (text-based, not actual tables)
3. Scanned docs: OCR can merge/split table cells incorrectly
4. Multi-page tables (continuation across pages)
5. Sub-line items (variants, specifications nested under main item)

APPROACH: TABLE DETECTION → STRUCTURE RECOGNITION → CELL EXTRACTION

┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Table Detection                                          │
│ - Object detection model (fine-tuned DETR) identifies table regions│
│ - Handles: bordered tables, borderless tables, form-style layouts │
│ - Output: bounding boxes of table regions on each page            │
│                                                                   │
│ STEP 2: Structure Recognition                                     │
│ - Identify rows, columns, headers                                 │
│ - Handle multi-line rows (description wrapping)                   │
│ - Handle spanning cells (merged)                                  │
│ - Output: grid structure with cell bounding boxes                 │
│                                                                   │
│ STEP 3: Cell Content Extraction                                   │
│ - OCR each cell (or extract text if digital PDF)                  │
│ - Classify each column: SKU | description | quantity | unit_price │
│   | line_total | UOM | tax | discount                             │
│ - Normalize: "1,234.56" → 1234.56, "12 EA" → {qty: 12, uom: EA} │
│                                                                   │
│ STEP 4: Semantic Matching                                         │
│ - Match extracted SKU to our product catalog (fuzzy matching)     │
│ - Match description to known products (embedding similarity)      │
│ - Resolve ambiguity: "Milk 1L" → which of 15 milk SKUs?          │
│   → Use context: vendor, price, historical orders                  │
└─────────────────────────────────────────────────────────────────┘

ARITHMETIC VALIDATION (catches ~40% of extraction errors):
- For each line: qty × unit_price should = line_total (±$0.01)
- Sum of line_totals + tax + shipping should = invoice total
- If math doesn't check out → flag specific cells for review
- This is a FREE accuracy boost — no ML needed, just arithmetic

MULTI-PAGE TABLE HANDLING:
- Detect: header repeats at top of page 2 → continuation
- Merge: concatenate rows, re-validate structure
- Challenge: last row of page 1 + first row of page 2 might be same item
- Solution: check if row is "complete" (has qty, price, total) before closing
"
```

---

### Deep Dive 3: Learning from Corrections (Active Learning)

```
"The system should get better over time. Here's the learning architecture:

┌─────────────────────────────────────────────────────────────────┐
│            ACTIVE LEARNING LOOP                                   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ CORRECTION CAPTURE                                        │    │
│  │                                                           │    │
│  │ When human corrects a field:                              │    │
│  │ Store: {                                                  │    │
│  │   document_id, page_image, field_name,                    │    │
│  │   predicted_value, correct_value,                         │    │
│  │   model_used, confidence_was, template_id                 │    │
│  │ }                                                         │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ ANALYSIS (nightly batch)                                  │    │
│  │                                                           │    │
│  │ 1. Error clustering:                                      │    │
│  │    - Group corrections by pattern (same field, same       │    │
│  │      template, same error type)                           │    │
│  │    - "Template X always gets vendor name wrong"           │    │
│  │    - "Date format DD/MM/YYYY confused with MM/DD/YYYY"    │    │
│  │                                                           │    │
│  │ 2. Template discovery:                                    │    │
│  │    - Cluster documents that went to VLM (expensive path)  │    │
│  │    - If cluster > 10 docs with similar layout → new template│   │
│  │    - Auto-create template, fine-tune extraction model     │    │
│  │                                                           │    │
│  │ 3. Confidence calibration:                                │    │
│  │    - Is confidence score well-calibrated?                 │    │
│  │    - "99% confidence" should mean 99% are actually correct│    │
│  │    - Adjust thresholds based on observed accuracy         │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ RETRAINING (weekly, automated)                            │    │
│  │                                                           │    │
│  │ 1. Add corrected samples to training set                  │    │
│  │ 2. Fine-tune template models on expanded data             │    │
│  │ 3. Evaluate on held-out test set (MUST improve, not regress)│  │
│  │ 4. Shadow deployment: run new model alongside old         │    │
│  │ 5. If new model better on all metrics → promote           │    │
│  │ 6. If regression on any template → investigate before deploy│  │
│  │                                                           │    │
│  │ GUARD RAILS:                                              │    │
│  │ - Never auto-deploy model worse than current on ANY metric│    │
│  │ - Minimum training samples per template: 50               │    │
│  │ - Human ML engineer reviews weekly retrain results        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  EXPECTED IMPROVEMENT CURVE:                                      │
│  - Week 1: 85% auto-approval rate (cold start)                   │
│  - Month 1: 90% (learned common templates)                       │
│  - Month 3: 94% (long tail of templates captured)                │
│  - Month 6: 96% (refinements, edge cases)                        │
│  - Steady state: 95-97% (new vendors/formats always arriving)    │
└─────────────────────────────────────────────────────────────────┘
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

### Three-Way Document Matching
```
"A critical downstream use case: matching PO → Invoice → Receipt

Why: Detect discrepancies (overcharging, short shipments, unauthorized purchases)

PO says: 100 units of SKU-789 at $45/unit = $4,500
Invoice says: 100 units of SKU-789 at $47/unit = $4,700  ← PRICE MISMATCH
Receipt says: 95 units of SKU-789  ← QUANTITY MISMATCH

The document processing system extracts structured data that enables 
this matching. The matching engine is a separate service that:
1. Correlates documents by PO number
2. Compares field values across documents
3. Flags discrepancies above threshold
4. Routes to AP team for resolution

This is where the real business value lives — not just in extraction,
but in the automated reconciliation it enables."
```

### Key Trade-offs:
```
"1. ACCURACY vs THROUGHPUT:
   - Could achieve 99.9% with 3 models + human review on everything
   - But that defeats the purpose (saving human time)
   - My choice: tiered confidence → auto-approve high-confidence, 
     human-review low-confidence only

2. GENERALIZATION vs SPECIALIZATION:
   - VLM generalizes to any document (expensive, slower)
   - Template model specializes per format (cheap, fast, limited)
   - My choice: specialize where possible, generalize where necessary

3. LEARNING SPEED vs STABILITY:
   - Could retrain daily for fastest learning
   - But risks instability (bad batch of corrections → model regression)
   - My choice: weekly retrain with shadow evaluation before deployment"
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Pipeline design | Clear stages: preprocess → classify → extract → validate | ☐ |
| Multi-model strategy | Tiered by cost/accuracy, intelligent routing | ☐ |
| Table extraction | Specific approach for the hardest part (line items) | ☐ |
| Validation | Arithmetic checks, referential integrity, business rules | ☐ |
| Human-in-the-loop | Smart routing, efficient UI, minimal reviewer burden | ☐ |
| Active learning | Correction capture, clustering, automated retraining | ☐ |
| Business value | 3-way matching, auto-approval rate, cost savings | ☐ |
| Staff signals | Phased rollout, metrics-driven, operational concerns | ☐ |


---


# Mock Interview 6: Supply Chain Disruption Prediction System (Moonshot)

## Interview Format: 45-minute System Design (Staff Level) — Novel Problem

---

## The Prompt (What the Interviewer Says)

> "Design a system that predicts supply chain disruptions before they happen — anywhere from hours to weeks in advance. This could be anything: a factory fire, a port closure, a pandemic, a geopolitical event, a supplier bankruptcy, extreme weather. The system should assess risk, predict impact, and recommend preemptive actions. This is a hard, open-ended problem — think big."

---

## Why This is a "Moonshot" Question

This question has no standard answer. The interviewer is evaluating:
1. How you decompose an impossibly broad problem into tractable pieces
2. How you handle deep uncertainty (many disruptions are unprecedented)
3. Whether you can think in first principles vs just reciting patterns
4. Your ability to propose novel approaches while being honest about limitations

---

## PHASE 1: Clarifying Questions & Problem Decomposition (8 minutes)

### Questions YOU Should Ask:

**Scoping the Problem:**
1. "Let me decompose 'supply chain disruptions' — I see at least 5 categories with very different prediction approaches. Should I design for all, or focus on a subset?"
   ```
   Category 1: Weather (hurricanes, floods, heat waves)
     → Predictability: 3-14 days ahead, high confidence
     
   Category 2: Geopolitical (sanctions, tariffs, war)
     → Predictability: signals weeks ahead, high uncertainty
     
   Category 3: Supplier-specific (bankruptcy, quality failure, fire)
     → Predictability: some financial signals, mostly surprise
     
   Category 4: Infrastructure (port congestion, transport strike)
     → Predictability: days to weeks, depends on root cause
     
   Category 5: Demand shock (viral trend, competitor exit, pandemic)
     → Predictability: hours to days after onset, hard to predict onset
   ```

2. "Who is the customer for this system? A single company protecting its supply chain, or a platform serving many companies?"
3. "What's the prediction horizon that matters most — 24 hours? 1 week? 1 month?"
4. "What constitutes a 'disruption'? Any delay? Or only significant impacts (>$X damage)?"
5. "How do we validate predictions? Some disruptions are prevented (survivorship bias), and some are so rare we can't train on them."

**About Capabilities:**
6. "Can we acquire external data? (News APIs, satellite imagery, financial data, social media, weather)"
7. "Do we have access to the company's supply chain graph? (Who supplies what, through which routes)"
8. "What's the tolerance for false positives? Alert fatigue is a real risk."
9. "Budget for data acquisition? (Satellite feeds, premium news, financial data = expensive)"

### Interviewer's Likely Answer:
- All categories in scope (but prioritize by feasibility)
- Platform serving many companies (multi-tenant)
- 1-14 day prediction horizon is most actionable
- Significant disruptions only (>$100K potential impact)
- External data acquisition: yes, reasonable budget
- False positive tolerance: <20% (5 false alerts per 1 real disruption)
- Supply chain graph available per customer

---

## PHASE 2: First Principles Framing (5 minutes)

```
"Let me frame this from first principles. 

A supply chain disruption has three components:
1. A HAZARD (something bad happens somewhere in the world)
2. An EXPOSURE (a supply chain element is in the blast radius)
3. An IMPACT (the disruption propagates through the supply chain)

My system needs three corresponding capabilities:
1. HAZARD DETECTION: Detect or predict the bad event
2. EXPOSURE MAPPING: Know which supply chain elements are affected
3. IMPACT PROPAGATION: Model how disruption cascades through the network

Prediction difficulty varies by category:

                    Predictable ←────────→ Unpredictable
                    │                              │
   Weather ─────────┤                              │
   Infrastructure ──┤                              │
   Geopolitical ────────────────┤                  │
   Supplier failure ─────────────────────┤         │
   Black swans ────────────────────────────────────┤

For predictable categories: BUILD PREDICTION MODELS
For unpredictable categories: BUILD EARLY DETECTION + RAPID RESPONSE

This is NOT just an ML problem. It's a multi-source intelligence 
system with human-in-the-loop judgment."
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    SIGNAL COLLECTION LAYER                               │
│                                                                          │
│  ┌───────────┐ ┌────────────┐ ┌───────────┐ ┌─────────┐ ┌──────────┐│
│  │ News &    │ │ Weather &  │ │ Financial │ │ Social  │ │ Satellite││
│  │ Media     │ │ Climate    │ │ Markets   │ │ Media   │ │ & IoT    ││
│  │ (100K     │ │ (forecasts,│ │ (supplier │ │ (Twitter│ │ (port    ││
│  │  articles/│ │  satellite,│ │  stock,   │ │  Reddit,│ │  imagery,││
│  │  day)     │ │  sensors)  │ │  credit   │ │  forums)│ │  AIS ship││
│  │           │ │            │ │  ratings) │ │         │ │  tracking││
│  └─────┬─────┘ └─────┬──────┘ └─────┬─────┘ └────┬────┘ └─────┬────┘│
└────────┼──────────────┼──────────────┼────────────┼────────────┼─────┘
         │              │              │            │            │
         ▼              ▼              ▼            ▼            ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  SIGNAL PROCESSING LAYER                                 │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ NLP PIPELINE (for text sources)                                  │    │
│  │ - Entity extraction: companies, locations, events                │    │
│  │ - Event detection: "fire at factory", "strike announced"         │    │
│  │ - Sentiment: escalation detection (getting worse over time?)     │    │
│  │ - Geo-coding: map events to locations                            │    │
│  │ - Deduplication: same event from multiple sources                │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ ANOMALY DETECTION (for numeric/spatial sources)                  │    │
│  │ - Weather: severe weather alerts exceeding thresholds            │    │
│  │ - Financial: credit score drops, stock price crashes             │    │
│  │ - Satellite: port congestion (vessel count above normal)         │    │
│  │ - IoT: unusual patterns at facilities                            │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  OUTPUT: Normalized SIGNAL objects                                       │
│  {signal_id, type, severity, location, affected_entities,               │
│   confidence, source, timestamp, predicted_duration}                     │
└─────────────────────────────────────────────────────┬────────────────────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                KNOWLEDGE GRAPH & EXPOSURE MAPPING                        │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ SUPPLY CHAIN KNOWLEDGE GRAPH (per customer)                     │    │
│  │                                                                  │    │
│  │ Nodes:                                                           │    │
│  │   [Suppliers] → [Materials] → [Factories] → [Products]          │    │
│  │   [Ports] → [Routes] → [Warehouses] → [Customers]               │    │
│  │                                                                  │    │
│  │ Node attributes:                                                 │    │
│  │   - Geographic coordinates (for spatial hazard matching)         │    │
│  │   - Criticality score (single-source? alternative exists?)       │    │
│  │   - Historical reliability                                       │    │
│  │   - Financial health indicators                                  │    │
│  │                                                                  │    │
│  │ Edges:                                                           │    │
│  │   - "supplies" (with volume, lead time, contract terms)          │    │
│  │   - "routes through" (mode, typical transit time)                │    │
│  │   - "alternative to" (substitution possibility)                  │    │
│  │                                                                  │    │
│  │ EXPOSURE QUERY:                                                  │    │
│  │   Signal: "Port of Shanghai congestion"                          │    │
│  │   Query: "Which of my supply chains route through Shanghai?"     │    │
│  │   Result: 47 active shipments, 12 suppliers, affecting 200 SKUs  │    │
│  │   Criticality: 3 are single-source materials (HIGH RISK)         │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┬────────────────────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                RISK ASSESSMENT & PREDICTION ENGINE                       │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ RISK SCORING MODEL                                               │    │
│  │                                                                  │    │
│  │ Risk Score = P(disruption) × Severity × Exposure × (1/Resilience)│   │
│  │                                                                  │    │
│  │ P(disruption): probability event causes actual supply disruption │    │
│  │   - Derived from: signal severity, historical base rates,        │    │
│  │     geographic proximity, duration prediction                    │    │
│  │                                                                  │    │
│  │ Severity: if disrupted, how bad is it?                           │    │
│  │   - Duration estimate (1 day vs 1 month)                         │    │
│  │   - Capacity reduction (10% slowdown vs 100% shutdown)           │    │
│  │                                                                  │    │
│  │ Exposure: how much of my supply chain is in the blast radius?    │    │
│  │   - Number of affected nodes                                     │    │
│  │   - Dollar value at risk                                         │    │
│  │   - SKU criticality (stockout impact)                            │    │
│  │                                                                  │    │
│  │ Resilience: how well can we absorb this?                         │    │
│  │   - Alternative suppliers available?                              │    │
│  │   - Safety stock coverage?                                       │    │
│  │   - Alternative routes exist?                                    │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ PREDICTION MODELS (per disruption category)                      │    │
│  │                                                                  │    │
│  │ Weather Prediction:                                              │    │
│  │   Input: NOAA forecasts, satellite data, historical patterns     │    │
│  │   Output: P(severe weather at location X in next N days)         │    │
│  │   Method: Ensemble of meteorological models + our logistics impact│   │
│  │                                                                  │    │
│  │ Supplier Financial Health:                                       │    │
│  │   Input: Stock price, credit ratings, payment behavior,          │    │
│  │          job postings, news sentiment, Glassdoor reviews         │    │
│  │   Output: P(supplier distress in next 90 days)                   │    │
│  │   Method: Gradient boosted classifier on financial indicators    │    │
│  │                                                                  │    │
│  │ Geopolitical Risk:                                               │    │
│  │   Input: News volume/sentiment, diplomatic events, sanctions     │    │
│  │   Output: Risk score per country/region (relative, not absolute) │    │
│  │   Method: NLP + time-series on news features (HARD, low accuracy)│    │
│  │                                                                  │    │
│  │ Infrastructure:                                                  │    │
│  │   Input: Port AIS data, congestion history, labor news           │    │
│  │   Output: P(port delay > 3 days in next week)                    │    │
│  │   Method: Time-series forecasting on congestion metrics          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┬────────────────────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  IMPACT SIMULATION & RECOMMENDATION                      │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ PROPAGATION MODEL                                                │    │
│  │                                                                  │    │
│  │ Given: disruption at node X with severity S and duration D       │    │
│  │ Simulate: cascade through supply chain graph                     │    │
│  │                                                                  │    │
│  │ Example:                                                         │    │
│  │   "Supplier A shut down for 2 weeks"                             │    │
│  │     → Material M unavailable                                     │    │
│  │       → Product P cannot be manufactured after day 5 (buffer runs out) │
│  │         → 3 customers affected, $2.3M revenue at risk            │    │
│  │         → Alternative supplier B can supply in 10 days (partial) │    │
│  │                                                                  │    │
│  │ Method: Discrete event simulation on supply chain graph          │    │
│  │ - Consume existing inventory buffers first                       │    │
│  │ - Model production dependencies (BOM explosion)                  │    │
│  │ - Track which end customers are affected and when                │    │
│  │ - Monte Carlo: simulate with uncertainty in duration             │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ RECOMMENDATION ENGINE                                            │    │
│  │                                                                  │    │
│  │ Possible actions (ordered by cost/reversibility):                │    │
│  │ 1. MONITOR: Do nothing yet, watch for escalation                 │    │
│  │ 2. ALERT: Notify stakeholders, prepare contingency               │    │
│  │ 3. PRE-ORDER: Place safety orders with alternative suppliers     │    │
│  │ 4. REROUTE: Redirect in-transit shipments                        │    │
│  │ 5. EXPEDITE: Pay for faster shipping on critical items           │    │
│  │ 6. SUBSTITUTE: Use alternative product/material                  │    │
│  │ 7. COMMUNICATE: Proactively inform affected customers            │    │
│  │                                                                  │    │
│  │ Decision logic:                                                  │    │
│  │ if risk_score > CRITICAL_THRESHOLD and action_cost < impact_cost:│    │
│  │     recommend_action(preemptive=True)                            │    │
│  │ elif risk_score > HIGH_THRESHOLD:                                │    │
│  │     recommend_action(prepare_contingency=True)                   │    │
│  │ else:                                                            │    │
│  │     monitor_and_reassess(frequency=6_hours)                      │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Multi-Source Signal Fusion

```
"The core challenge: How do you combine signals from news, weather, 
financial data, and satellite imagery into a coherent risk assessment?

THE SIGNAL FUSION PROBLEM:
- Same event appears in multiple sources with different details
- Different sources have different latencies (Twitter fast, news slow, 
  financial data very slow)
- Different sources have different reliability
- False signals are common (news is sensationalized, social media is noisy)

MY APPROACH: Event-Centric Fusion

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Step 1: EVENT CANDIDATE GENERATION                               │
│  - Each source independently generates 'event candidates'        │
│  - A candidate = {type, location, severity, confidence, source}  │
│                                                                   │
│  Step 2: EVENT CORRELATION                                        │
│  - Cluster candidates by: location proximity + time window + type │
│  - "Factory fire in Shenzhen" from news + "thermal anomaly in    │
│    Shenzhen industrial zone" from satellite = SAME EVENT          │
│  - Use entity resolution: "Foxconn" + "Hon Hai" = same company   │
│                                                                   │
│  Step 3: CONFIDENCE SCORING                                       │
│  - Single-source event: confidence = source_reliability × 0.5    │
│  - Multi-source corroborated: confidence boosted proportionally   │
│  - Contradicting sources: flag for investigation                  │
│                                                                   │
│  Confidence calculation:                                          │
│  P(real event) = 1 - Π(1 - P(source_i_correct))                 │
│                                                                   │
│  Example:                                                         │
│  - News source (70% reliable) reports port congestion             │
│  - Satellite confirms (85% reliable) high vessel density          │
│  - P(real) = 1 - (0.3 × 0.15) = 1 - 0.045 = 95.5%              │
│                                                                   │
│  Step 4: SEVERITY ESTIMATION                                      │
│  - Use historical analogies: "Similar events in the past caused   │
│    X days of disruption with Y% probability"                     │
│  - Adjust for specifics: scale, affected entity, context          │
│  - Output: probability distribution over severity levels          │
│                                                                   │
│  Step 5: TEMPORAL TRACKING                                        │
│  - Events evolve over time (strike announced → negotiations →     │
│    strike begins → resolution)                                   │
│  - Track event lifecycle with state machine:                      │
│    DETECTED → CONFIRMED → ESCALATING → PEAK → RESOLVING → RESOLVED │
│  - Update risk score as event progresses                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

HANDLING NOISE (the hardest part):
- News mentions 'supply chain' 50K times/day. 99.9% is noise.
- Multi-stage filtering:
  1. Relevance filter (LLM classifier): "Is this about a specific 
     disruption, or just commentary?" → drops 95%
  2. Entity filter: "Does this mention an entity in our customer's 
     supply chain?" → drops 80% of remainder
  3. Severity filter: "Is this significant enough to matter?" → drops 50%
  4. Result: ~50 actionable signals per day from 50K raw mentions
"
```

---

### Deep Dive 2: Handling Prediction for Rare Events

```
"The fundamental challenge: truly disruptive events are RARE. 
How do you train models for events that happen once every 10 years?

THE RARITY PROBLEM:
- Pandemics: ~3 in 100 years
- Port closures: ~5-10/year globally
- Supplier bankruptcies: varies, but specific ones are unpredictable
- We can't train on what hasn't happened

MY APPROACH: Multi-Strategy for Different Predictability Levels

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  STRATEGY 1: ANALOG-BASED REASONING (for unprecedented events)   │
│  ──────────────────────────────────────────────────────────────  │
│  Can't predict Black Swans, but can reason about impact:          │
│                                                                   │
│  "We've never seen a pandemic close Chinese factories before,     │
│   but we CAN model: 'If Chinese factories are closed for X weeks, │
│   what happens to our supply chain?'"                             │
│                                                                   │
│  → SCENARIO-BASED: Pre-compute impact for hypothetical disruptions│
│    - "What if Supplier A goes down for 2 weeks?"                  │
│    - "What if Port X closes for 1 month?"                         │
│    - Generate a library of impact scenarios                       │
│    - When real event occurs → match to closest scenario           │
│                                                                   │
│  STRATEGY 2: LEADING INDICATORS (for predictable categories)      │
│  ──────────────────────────────────────────────────────────────  │
│  Some disruptions have reliable precursors:                       │
│                                                                   │
│  Supplier bankruptcy (30-90 day warning signs):                   │
│    - Stock price decline > 30% in 90 days                        │
│    - Credit rating downgrade                                      │
│    - Payment behavior change (paying invoices late)               │
│    - Layoff announcements / hiring freeze                         │
│    - Key executive departures                                     │
│    - Negative news sentiment acceleration                         │
│                                                                   │
│  Port congestion (3-7 day warning):                              │
│    - Vessel queue length increasing                               │
│    - Average wait time trending up                                │
│    - Nearby ports seeing overflow traffic                         │
│    - Labor union rhetoric escalating                              │
│                                                                   │
│  STRATEGY 3: TRANSFER LEARNING FROM ADJACENT DOMAINS              │
│  ──────────────────────────────────────────────────────────────  │
│  - Insurance industry has extensive catastrophe models            │
│  - Climate science has weather prediction models                  │
│  - Epidemiology has disease spread models                         │
│  - Financial markets are leading indicators for economic shocks   │
│  → Integrate these external models as features                    │
│                                                                   │
│  STRATEGY 4: HUMAN INTELLIGENCE AUGMENTATION                     │
│  ──────────────────────────────────────────────────────────────  │
│  For truly novel events, AI alone is insufficient:               │
│  - Industry analysts provide qualitative risk assessments         │
│  - On-ground contacts at key locations provide local intel        │
│  - "Prediction markets" among supply chain professionals          │
│  - LLM-powered synthesis: "Given these 10 signals, what's        │
│    your assessment?" → structured by AI, judged by human          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

HONEST LIMITATIONS (say this in the interview):
"Let me be explicit about what this system CANNOT do:
1. Predict Black Swans (by definition, unprecedented events are unpredictable)
2. Assign precise probabilities to rare events (not enough training data)
3. Replace human judgment for geopolitical risk (too complex)

What it CAN do:
1. Detect disruptions faster than humans monitoring manually
2. Instantly assess exposure and impact (graph traversal)
3. Quantify the blast radius before it cascades
4. Recommend preemptive actions that reduce impact
5. Learn from each disruption to detect similar patterns next time"
```

---

### Deep Dive 3: Evaluation — How Do You Know This System Works?

```
"This is a critical question for any prediction system, especially one 
predicting rare events.

THE EVALUATION CHALLENGE:
- Can't wait for rare events to measure accuracy (too slow)
- Prevented disruptions are invisible (survivorship bias)
- False positives are costly (erode trust) but false negatives are 
  catastrophic (miss real disruptions)

MY EVALUATION FRAMEWORK:

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  LEVEL 1: COMPONENT EVALUATION (continuous)                      │
│  - Signal detection precision/recall per source                  │
│  - Entity resolution accuracy (are we matching events correctly?)│
│  - Severity estimation calibration (are our probabilities real?) │
│  - Exposure mapping coverage (do we know about all supply paths?)│
│                                                                   │
│  LEVEL 2: BACKTESTING (monthly)                                  │
│  - Replay historical disruptions through the system              │
│  - "Given data available BEFORE event X, would we have detected  │
│    it, and how early?"                                           │
│  - Use: COVID factory closures (Jan 2020), Suez Canal (Mar 2021),│
│    Texas freeze (Feb 2021), Shanghai lockdown (Apr 2022)         │
│  - Measure: detection lead time, severity accuracy, action quality│
│                                                                   │
│  LEVEL 3: SIMULATED DISRUPTIONS (quarterly)                      │
│  - Inject synthetic signals that simulate a disruption building  │
│  - Does the system detect, assess, and recommend correctly?      │
│  - Like fire drills for the prediction system                    │
│  - Red team: try to fool the system with misleading signals      │
│                                                                   │
│  LEVEL 4: REAL-WORLD OUTCOMES (ongoing tracking)                 │
│  - Track every alert issued: was the prediction correct?         │
│  - Track every disruption that occurred: did we alert in advance?│
│  - Key metrics:                                                   │
│    - Detection rate: % of real disruptions we alerted on         │
│    - Lead time: how far in advance did we alert?                 │
│    - Precision: % of alerts that corresponded to real events     │
│    - Action value: $ saved by preemptive actions taken            │
│                                                                   │
│  TARGET METRICS:                                                  │
│  - Detection rate: >80% of significant disruptions (week+ ahead) │
│  - Lead time: >3 days average warning before impact              │
│  - Precision: >80% (4 real alerts per 1 false)                   │
│  - Action value: measurable $ savings from early response        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

### Evolution & What to Build First:
```
"Given the difficulty spectrum, I'd phase this:

Phase 1 (Month 1-3): REACTIVE INTELLIGENCE
  - Rapid detection of KNOWN disruption patterns
  - Instant exposure mapping (who's affected?)
  - Manual risk assessment, automated impact simulation
  - Value: detect faster than humans scanning news

Phase 2 (Month 3-6): PREDICTIVE WEATHER & LOGISTICS
  - Weather-based disruption prediction (most tractable)
  - Port congestion forecasting (good data available)
  - Automated severity estimation
  - Value: 3-7 day advance warning for weather/logistics

Phase 3 (Month 6-12): SUPPLIER RISK SCORING
  - Financial health monitoring
  - Leading indicator models for supplier distress
  - Value: 30-90 day early warning on supplier failures

Phase 4 (12+ months): FULL PREDICTIVE PLATFORM
  - Geopolitical risk (hardest, lowest accuracy)
  - Novel event detection (pattern matching to historical analogs)
  - Autonomous preemptive actions
  - Value: comprehensive risk coverage

THE FIRST THING I'D BUILD (week 1):
A simple prototype that:
1. Monitors 5 news APIs for keywords matching our customer's top 20 suppliers
2. If match → query supply chain graph → show affected products
3. Human validates: 'Was this alert useful?'
4. Learn from 2 weeks of human feedback before building anything complex

This validates the MOST IMPORTANT ASSUMPTION: can we detect supply chain-
relevant events from public signals in a timely and useful way?
If this doesn't work, the whole system concept fails."
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Problem decomposition | Categorized disruption types, different approaches per type | ☐ |
| First principles | Risk = P(hazard) × Exposure × Impact ÷ Resilience | ☐ |
| Multi-source fusion | Correlation, confidence scoring, noise filtering | ☐ |
| Rare event handling | Analogs, scenarios, leading indicators, honest about limits | ☐ |
| Knowledge graph | Supply chain as graph, exposure query, cascade propagation | ☐ |
| Evaluation strategy | Backtesting, simulation, real-world tracking | ☐ |
| Honest about limitations | Explicitly stated what system CAN'T do | ☐ |
| Moonshot + pragmatic | Big vision but phased practical delivery | ☐ |
| Staff signals | "What I'd build first", de-risking assumptions | ☐ |
