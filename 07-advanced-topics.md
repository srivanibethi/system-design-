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
