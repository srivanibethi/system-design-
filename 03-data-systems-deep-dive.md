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
