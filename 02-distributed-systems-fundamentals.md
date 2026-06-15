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
