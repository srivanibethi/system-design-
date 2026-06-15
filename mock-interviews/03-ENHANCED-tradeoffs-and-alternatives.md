# ENHANCED: Real-Time Event Processing — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 3 — Read alongside the main file

---

## RIGOROUS SCALE ESTIMATES

### Event Volume Modeling
```
SOURCE BREAKDOWN (50M events/day target):
┌──────────────────────────────────────────────────────────────────┐
│ Source              │ Events/day │ Avg Size │ Pattern              │
├─────────────────────┼────────────┼──────────┼──────────────────────┤
│ Warehouse WMS       │ 15M        │ 400B     │ Steady during shifts │
│ Carrier tracking    │ 12M        │ 600B     │ Batchy (hourly dumps)│
│ POS/store events    │ 10M        │ 300B     │ Peaks at meal times  │
│ IoT sensors         │ 8M         │ 200B     │ Continuous, uniform  │
│ ERP changes (CDC)   │ 3M         │ 800B     │ Business hours heavy │
│ Supplier updates    │ 2M         │ 500B     │ Random, bursty       │
├─────────────────────┼────────────┼──────────┼──────────────────────┤
│ TOTAL               │ 50M        │ ~420B avg│                      │
└──────────────────────────────────────────────────────────────────┘

THROUGHPUT MATH:
- Average: 50M/day ÷ 86,400 = 580 events/sec
- Peak (Black Friday, shift change, carrier batch dump):
  - 10x average = 5,800 events/sec
  - Burst (carrier dumps 6 hours of data): 20x for 5 minutes = 11,600/sec
- Kafka can handle 1M+ events/sec per broker → COMFORTABLE

STORAGE:
- Raw events: 50M × 420B = 21 GB/day
- 90-day hot storage: 1.9 TB (BigQuery or Bigtable)
- 7-year cold archive: 55 TB (GCS, compressed: ~15 TB)
- Processed/enriched events: 2x raw size = 42 GB/day (metadata added)
- Aggregated metrics: <1 GB/day (time-series rollups)
```

### Anomaly Detection Scaling
```
DETECTION PIPELINE COMPUTE:

Layer 1 (Rules): 
  - 200 rules evaluated per event
  - CPU cost: ~10μs per event (simple comparisons)
  - 5,800 events/sec peak × 10μs = 58ms of CPU per second
  - 1 core handles this easily
  
Layer 2 (Statistical):
  - Per-entity Z-score requires state (running mean, stddev)
  - Entities: 100K SKUs × 500 locations = 50M entities
  - State per entity: 100 bytes (mean, variance, count, window)
  - Total state: 50M × 100B = 5 GB
  - Fits in Flink's RocksDB state backend
  - Cost: ~50μs per event (hash lookup + math)
  
Layer 3 (ML):
  - Isolation Forest inference: ~1ms per event batch (batch of 100)
  - 5,800 events/sec ÷ 100 batch = 58 inferences/sec
  - Single GPU: can do 10K+ inferences/sec → one GPU is enough
  - Sequence model: more expensive, ~5ms per sequence evaluation
  - Only trigger on suspicious entities (pre-filtered): ~500/sec
  - Cost: 1-2 GPUs

TOTAL COMPUTE FOR DETECTION:
  - 4-8 Flink TaskManagers (4 CPU, 16 GB each) for L1+L2
  - 2 GPU instances for L3 (can be preemptible for cost savings)
  - Cost: ~$5K/month compute
```

---

## ALTERNATIVE ARCHITECTURES

### Approach A: Lambda Architecture (Batch + Speed Layer)
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Events → Kafka → Speed Layer (Flink, ~1 min latency)            │
│                        ↓                                          │
│                   Real-time anomalies (approximate)                │
│                                                                   │
│  Events → Kafka → Batch Layer (Spark, nightly)                   │
│                        ↓                                          │
│                   Complete analysis (exact, all context)           │
│                                                                   │
│  Serving Layer merges both views                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

PROS:
+ Batch layer provides complete, accurate anomaly detection (full context)
+ Speed layer catches urgent issues fast
+ Batch layer can run complex ML models (no latency constraint)
+ Well-understood pattern, lots of tooling

CONS:
- Two code paths that must stay in sync (DRY violation)
- Operational complexity of maintaining Spark + Flink
- Batch results are delayed (anomaly detected 12 hours later is less useful)
- Logic bugs where speed and batch disagree

WHEN TO CHOOSE: When you have strong batch analytics needs ALONGSIDE real-time.
```

### Approach B: Kappa Architecture (Streaming Only) — MY CHOICE
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Events → Kafka → Flink (single processing path) → Results      │
│                                                                   │
│  For historical analysis: replay from Kafka beginning             │
│  For new algorithms: deploy new job, replay to catch up           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

PROS:
+ Single code path (simpler to maintain, no sync issues)
+ Lower operational complexity
+ Results are always real-time
+ Reprocessing = replay from Kafka (simple)
+ Modern Flink can handle complex analytics that used to require batch

CONS:
- Very complex ML models hard to run in streaming (latency constraints)
- Historical analysis requires replaying weeks of data (can be slow)
- State management is harder than batch (checkpointing, recovery)
- Flink expertise is rarer than Spark expertise

WHEN TO CHOOSE: When real-time is the primary requirement and you can
accept slightly simpler models in exchange for architectural simplicity.
```

### Approach C: Micro-Batch (Spark Structured Streaming)
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Events → Kafka → Spark Structured Streaming → Results           │
│                    (micro-batches every 10-30 seconds)            │
│                                                                   │
│  Same Spark codebase for batch AND streaming                     │
│  Unified API, familiar to most data engineers                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

PROS:
+ Familiar (most teams know Spark)
+ Same code for batch and streaming (truly unified)
+ Good for moderate latency requirements (30-sec windows)
+ Rich ML library integration (MLlib)
+ Easier state management than Flink

CONS:
- Minimum latency = micro-batch size (10-30 seconds, not sub-second)
- Higher processing latency than true streaming (Flink)
- Event-time processing less mature than Flink's
- Checkpoint recovery is slower than Flink
- Not ideal for complex event processing (CEP patterns)

WHEN TO CHOOSE: When team already has Spark expertise, latency requirement
is 30+ seconds, and you value unified batch/streaming over minimum latency.
```

### Approach D: Event-Driven Microservices (No Stream Processor)
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Events → Kafka → Consumer Services (custom Go/Python)           │
│                                                                   │
│  Service 1: Validator (consume, validate, produce to next topic) │
│  Service 2: Enricher (consume, lookup metadata, produce)         │
│  Service 3: Detector (consume, run anomaly logic, produce)       │
│  Service 4: Alerter (consume anomalies, send notifications)      │
│                                                                   │
│  Each service is a simple Kafka consumer with custom logic       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

PROS:
+ Each service is simple and independently deployable
+ No Flink/Spark expertise needed (standard backend engineering)
+ Easy to scale individual services independently
+ Language flexibility per service
+ Simpler debugging (each service has clear input/output)

CONS:
- No built-in windowing, state management, or event-time processing
- Must implement watermarks, dedup, checkpointing manually
- Exactly-once semantics very hard to achieve manually
- Windowed aggregations require custom state (Redis/DB)
- Much more glue code than using a stream processor
- Higher total engineering effort long-term

WHEN TO CHOOSE: Small team, latency < requirements < complexity of Flink,
or when the processing logic is simple (mostly routing and enrichment,
not complex windowed aggregations).
```

### My Architecture Decision Matrix:
```
┌─────────────────┬────────────┬──────────┬─────────┬──────────────┐
│ Requirement     │ Lambda (A) │ Kappa (B)│ Micro(C)│ Services (D) │
├─────────────────┼────────────┼──────────┼─────────┼──────────────┤
│ Latency <5 min  │ ✓          │ ✓✓       │ ✓       │ ✓            │
│ Complex windowed│ ✓✓         │ ✓✓       │ ✓       │ ✗ (manual)   │
│ ML integration  │ ✓✓ (batch) │ ✓        │ ✓       │ ✓            │
│ Ops simplicity  │ ✗          │ ✓        │ ✓✓      │ ✓ (per svc)  │
│ Team expertise  │ Spark      │ Flink    │ Spark   │ Any backend  │
│ Event-time proc │ ✓          │ ✓✓       │ ✓       │ ✗ (manual)   │
│ Exactly-once    │ ✓ (complex)│ ✓✓       │ ✓       │ ✗ (very hard)│
│ Reprocessing    │ ✓✓         │ ✓        │ ✓✓      │ ✗            │
├─────────────────┼────────────┼──────────┼─────────┼──────────────┤
│ OVERALL         │ Complex but│ Best for │ Good if │ Avoid for    │
│                 │ complete   │ real-time│ team fit│ this use case│
└─────────────────┴────────────┴──────────┴─────────┴──────────────┘

MY CHOICE: Kappa (B) with Flink because:
1. Primary requirement is REAL-TIME detection (batch is secondary)
2. Need complex windowed aggregations (tumbling + sliding + session)
3. Need event-time processing with watermarks (out-of-order events)
4. Flink's exactly-once with Kafka is battle-tested
5. Single code path reduces operational burden vs Lambda
6. ML inference can be done as an async side-output (doesn't block main pipeline)
```

---

## KAFKA TOPOLOGY DESIGN

### Topic Design Trade-offs
```
OPTION A: Single Topic (simple)
  raw-events (all event types in one topic)
  
  Pro: Simple consumers, easy to get complete picture
  Con: Mixed event types, consumers must filter, hard to scale by type

OPTION B: Topic Per Source (by origin)
  warehouse-events, carrier-events, pos-events, iot-events, erp-events
  
  Pro: Source-specific processing, independent retention
  Con: Cross-source correlation requires reading multiple topics

OPTION C: Topic Per Processing Stage (by pipeline step) — MY CHOICE
  raw-events → validated-events → enriched-events → anomalies → actions
  
  Pro: Clear pipeline stages, consumers at each stage independent
  Con: More topics to manage, data duplication across topics

MY CHOICE: C + sub-topics within each stage
  raw/warehouse, raw/carrier, raw/pos, raw/iot, raw/erp
  validated/all (unified after normalization)
  enriched/all (unified with metadata)
  anomalies/critical, anomalies/high, anomalies/medium
  actions/alerts, actions/auto-remediation

WHY: Pipeline stages give clear debugging points ("event stuck at 
enrichment stage"), and severity-split anomaly topics allow different
consumer SLAs (critical = pager, medium = daily digest).
```

### Partition Strategy
```
PARTITIONING TRADE-OFFS:

By entity_id (e.g., SKU-location hash):
  Pro: All events for one entity on same partition → ordered
  Con: Hot entities cause partition hotspots

By timestamp (round-robin with time salt):
  Pro: Even distribution, no hotspots
  Con: No ordering guarantee for same entity

By source_id (e.g., warehouse_id):
  Pro: All events from one warehouse together → easy local state
  Con: Uneven (large warehouse has 10x events of small one)

MY CHOICE: By entity_id for validated-events onward (anomaly detection
needs ordered events per entity), round-robin for raw-events (even load
at ingestion).

PARTITION COUNT:
  - 50 partitions for main topics
  - Why 50? 
    Peak: 5,800 events/sec ÷ 50 = 116 events/sec per partition
    Kafka recommendation: <1000 events/sec per partition → comfortable
    Consumer parallelism: 50 partitions = up to 50 consumer instances
    Not too many: >100 partitions increases controller overhead
```

---

## ANOMALY DETECTION TRADE-OFFS

### Detection Method Comparison
```
┌──────────────────────────────────────────────────────────────────────────┐
│ Method          │ Precision │ Recall │ Latency │ State Needed │ Adaptability│
├──────────────────┼───────────┼────────┼─────────┼──────────────┼─────────────┤
│ Fixed threshold │ Low       │ Low    │ 0ms     │ None         │ None        │
│ (temp > 40°C)   │ (many FP) │(miss  │         │              │ (static)    │
│                 │           │subtle) │         │              │             │
├──────────────────┼───────────┼────────┼─────────┼──────────────┼─────────────┤
│ Z-score         │ Medium    │ Medium │ 0ms     │ Mean, StdDev │ Slow        │
│ (|x-μ|/σ > 3)  │           │        │         │ per entity   │ (decays)    │
├──────────────────┼───────────┼────────┼─────────┼──────────────┼─────────────┤
│ CUSUM           │ High      │ High   │ Minutes │ Cumulative   │ Medium      │
│ (cumulative sum)│(for trend)│(trends)│ (needs  │ sum per      │ (resets on  │
│                 │           │        │  window)│ entity       │  detection) │
├──────────────────┼───────────┼────────┼─────────┼──────────────┼─────────────┤
│ Seasonal decomp │ High      │ Medium │ Hours   │ Full season  │ Good        │
│ (STL + residual)│           │        │ (needs  │ history      │ (relearns   │
│                 │           │        │  cycle) │              │  seasonality│
├──────────────────┼───────────┼────────┼─────────┼──────────────┼─────────────┤
│ Isolation Forest│ High      │ High   │ Seconds │ Trained model│ Requires    │
│ (ML, batch)     │           │        │ (batch  │ (retrain     │ retraining  │
│                 │           │        │  window)│  weekly)     │             │
├──────────────────┼───────────┼────────┼─────────┼──────────────┼─────────────┤
│ Autoencoder     │ High      │ Highest│ Seconds │ Trained model│ Online      │
│ (reconstruction)│           │        │ (infer) │ (GPU needed) │ fine-tuning │
│                 │           │        │         │              │ possible    │
└──────────────────────────────────────────────────────────────────────────────┘

MY TIERED SELECTION:
- Tier 1 (immediate): Fixed thresholds + Z-score (catches 60% of anomalies)
- Tier 2 (minutes): CUSUM for trend detection (catches 25% more)
- Tier 3 (near-real-time): Isolation Forest on 5-min windows (catches 10% more)
- Remaining 5%: too subtle, relies on downstream business metrics to catch

WHY THIS ORDER:
- Cheap methods first (filter 85% of normal data before expensive ML runs)
- Expensive models only see pre-filtered data (saves 10x compute)
- Each tier has different false-positive characteristics:
  - Thresholds: rarely false positive (configured conservatively)
  - Z-score: FP during seasonality changes (known, mitigated)
  - CUSUM: FP on legitimate gradual trends (need classification)
  - Isolation Forest: FP on unusual-but-correct patterns (need feedback loop)
```

### Correlation Engine Trade-offs
```
PROBLEM: 20 shipment delays ≠ 20 independent anomalies
         They might all be ONE carrier having a system problem

APPROACHES:

Option A: Time-Window Grouping (simple)
  If 3+ anomalies of same type within 30 minutes → group
  Pro: Simple, fast, no ML needed
  Con: Misses correlations across types or longer windows

Option B: Causal Graph (expert-defined)
  Pre-define: "carrier_delay → shipment_late → stockout_risk"
  When upstream event fires → proactively check downstream
  Pro: Precise, interpretable, catches cascades
  Con: Requires expert maintenance, doesn't discover new patterns

Option C: ML Clustering (learned)
  Cluster anomalies by: time proximity, entity graph distance, feature similarity
  Learn: which anomalies historically co-occur?
  Pro: Discovers non-obvious correlations, adapts
  Con: Opaque groupings, requires historical data, FP correlations

MY CHOICE: B (causal graph) + A (time-window) as fallback
  1. Define 50 known causal chains (from domain experts)
  2. When anomaly fires → walk causal graph → check related nodes
  3. Any ungrouped anomalies → simple time-window grouping
  4. Monthly review: which ungrouped anomalies co-occurred? → new causal edges

WHY NOT ML CLUSTERING:
  - At 500 alerts/day, there's not enough volume for statistical significance
  - False correlations would erode trust quickly
  - Causal graph is transparent (users can understand why alerts are grouped)
  - We can evolve the graph as we learn (human-in-the-loop)
```

---

## EXACTLY-ONCE PROCESSING: THE REAL TRADE-OFFS

```
┌─────────────────────────────────────────────────────────────────────┐
│ THE DIRTY SECRET: "Exactly-once" is an abstraction, not reality.     │
│ Under the hood, it's "at-least-once processing + idempotent output" │
│                                                                       │
│ FLINK'S EXACTLY-ONCE MECHANISM:                                      │
│ 1. Checkpoint barriers flow through the data stream                  │
│ 2. Each operator snapshots its state when barrier passes             │
│ 3. Kafka source commits offsets with checkpoint                      │
│ 4. On failure: restore from last checkpoint, replay from Kafka offset│
│                                                                       │
│ THE COST:                                                            │
│ - Checkpoint barriers add latency: 100-500ms per checkpoint          │
│ - Checkpoint interval trade-off:                                     │
│   - Short (10s): Low data loss on failure, high overhead             │
│   - Long (5min): Less overhead, more data to replay on failure       │
│   - MY CHOICE: 30s (balance: replay 30s on failure, moderate overhead)│
│                                                                       │
│ - State backend trade-off:                                           │
│   - Memory (HashMapStateBackend): Fastest, limited by RAM            │
│   - RocksDB: Slower, but handles state larger than memory            │
│   - MY CHOICE: RocksDB (50M entity states = 5GB, exceeds memory)    │
│                                                                       │
│ WHERE EXACTLY-ONCE MATTERS vs DOESN'T:                               │
│                                                                       │
│ MATTERS (use exactly-once):                                          │
│ - Counting anomalies (alert "5 delays in 1 hour" must be accurate)   │
│ - Financial aggregations (inventory value calculations)              │
│ - SLA violation counting (wrong count = contractual implications)    │
│                                                                       │
│ DOESN'T MATTER (at-least-once is fine):                              │
│ - Anomaly detection (detecting same anomaly twice = harmless, dedup)  │
│ - Alert generation (idempotent: same alert sent twice = ignore 2nd) │
│ - Logging/observability (duplicate log = no business impact)         │
│                                                                       │
│ MY DESIGN:                                                           │
│ - Main pipeline: exactly-once (for accurate metrics)                 │
│ - Alert side-output: at-least-once + idempotent alerts (faster)     │
│ - Logging: at-most-once (if we lose a log entry, life goes on)       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CAPACITY PLANNING FOR GROWTH

```
┌──────────────┬──────────┬──────────┬──────────┬───────────────────────┐
│              │ Year 0   │ Year 1   │ Year 2   │ Scaling Action        │
├──────────────┼──────────┼──────────┼──────────┼───────────────────────┤
│ Events/day   │ 5M       │ 50M      │ 200M     │ Add Kafka brokers     │
│ Peak QPS     │ 300      │ 5,800    │ 23,000   │ More Flink parallelism│
│ Entities     │ 1M       │ 50M      │ 200M     │ Shard state backend   │
│ Storage/day  │ 2 GB     │ 21 GB    │ 84 GB    │ Tiered (hot→warm→cold)│
│ Anomalies/day│ 5K       │ 50K      │ 200K     │ Better grouping/filter│
│ Alerts/day   │ 50       │ 500      │ 2000     │ ML severity, auto-act │
│ Kafka brokers│ 3        │ 6        │ 12       │ Linear with throughput│
│ Flink TMs    │ 2        │ 8        │ 20       │ Linear with QPS       │
│ Monthly cost │ $3K      │ $15K     │ $45K     │                       │
└──────────────┴──────────┴──────────┴──────────┴───────────────────────┘

KEY INSIGHT: Cost scales linearly with events, NOT with anomalies.
The anomaly detection FILTERS data — downstream (alerting, actions) 
handles a tiny fraction of the event volume.

At Year 2 (200M events/day):
- Ingestion + processing: $35K/month (scales with event volume)
- Anomaly detection: $5K/month (scales with entity count)
- Alerting + actions: $3K/month (scales with anomaly count — small!)
- Storage: $2K/month (tiered — most data in cold storage)
```
