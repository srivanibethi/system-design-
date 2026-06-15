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

### Expected Answers (Google-Scale):
- Google-scale multi-tenant platform: 1,000+ enterprise supply chains monitored simultaneously
- 5 BILLION events/day currently (across all tenants), growing to 50B within 18 months
- Mix of sources: APIs, webhooks, CDC streams, IoT sensors (100M+ devices), satellite feeds
- Detection within 30 seconds for critical anomalies, 5 minutes for standard
- Auto-actions from day one for well-understood patterns (reroute, reorder, alert)
- GCP-native, multi-region (us, eu, asia), leveraging Pub/Sub + Dataflow at extreme scale
- Massive out-of-order: IoT can be hours late, carrier feeds batchy, global timezone skew

---

## PHASE 2: Requirements Summary & Math (3 minutes)

```
"Let me quantify this at Google scale:

SCALE:
- Current: 5B events/day = ~58,000 events/sec avg
- Peak (Black Friday, Chinese NY, disruptions): 10x = 580,000 events/sec
- Burst (major global event — Suez canal, pandemic): 20x = 1.16M events/sec
- Target (18 mo): 50B events/day = 580K events/sec avg, 5.8M/sec peak
- This requires: Pub/Sub (unlimited) + Dataflow (auto-scaling) — Kafka alone won't work

EVENT SIZE:
- Avg event: 600 bytes (Protobuf with metadata, enrichment fields)
- Daily ingestion: 5B × 600B = 3 TB/day = 1.1 PB/year
- At 50B/day target: 30 TB/day = 11 PB/year
- Retention: 30 days hot (Bigtable), 1 year warm (BigQuery), 7 years cold (GCS)

ANOMALY DETECTION:
- 5B events/day, 0.1% anomalous = 5M potential anomalies/day
- After correlation + dedup: 500K unique anomaly patterns/day
- After severity filtering: 50K actionable alerts/day
- Per customer (1000 tenants): ~50 alerts/day avg (manageable)
- With 95% precision: 2,500 false positives/day across platform = 2.5/customer

ENTITY SCALE:
- 1000 tenants × avg 500K entities each = 500M entities tracked globally
- State per entity: running statistics, baselines, thresholds = 500 bytes
- Total stateful computation: 500M × 500B = 250 GB of streaming state
- This is LARGE for streaming — needs distributed state (Bigtable-backed)

LATENCY BUDGET (30-second SLO for critical):
- Event ingestion (Pub/Sub): <100ms
- Processing + enrichment: <5 sec
- Anomaly detection (streaming): <10 sec
- Correlation + severity: <10 sec
- Alert delivery: <5 sec
- Total: <30 sec end-to-end for critical path ✓
- Standard path (5 min SLO): allows batch ML models in the loop

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
# ENHANCED: Real-Time Event Processing — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 3 — Read alongside the main file

---

## RIGOROUS SCALE ESTIMATES (GOOGLE-SCALE)

### Event Volume Modeling
```
SOURCE BREAKDOWN (5B events/day — current, Google-scale platform):
┌──────────────────────────────────────────────────────────────────────────────┐
│ Source              │ Events/day  │ Avg Size │ Pattern                        │
├─────────────────────┼─────────────┼──────────┼────────────────────────────────┤
│ IoT sensors         │ 2B          │ 200B     │ Continuous, 100M+ devices      │
│ Warehouse WMS       │ 1B          │ 400B     │ Burst during shifts, 50K sites │
│ Carrier tracking    │ 800M        │ 600B     │ Batchy + streaming (GPS)       │
│ POS/store events    │ 500M        │ 300B     │ Peaks at meal/shopping times   │
│ ERP CDC streams     │ 400M        │ 800B     │ Business hours heavy           │
│ Supplier updates    │ 200M        │ 500B     │ Global, 24/7                   │
│ External (weather,  │ 100M        │ 1KB      │ Periodic pulls + push alerts   │
│  news, maritime)    │             │          │                                │
├─────────────────────┼─────────────┼──────────┼────────────────────────────────┤
│ TOTAL               │ 5B          │ ~600B avg│ 58K events/sec avg             │
└──────────────────────────────────────────────────────────────────────────────┘

THROUGHPUT MATH:
- Average: 5B/day ÷ 86,400 = 58,000 events/sec
- Peak (Black Friday + Chinese NY overlap, global disruption):
  - 10x average = 580,000 events/sec
  - Burst (cascading failures, major port closure): 20x = 1.16M events/sec
- At 50B/day target (18 months): 580K avg, 5.8M/sec peak

WHY KAFKA ALONE WON'T WORK AT THIS SCALE:
  - Kafka can handle 1M+/sec per cluster, BUT:
  - Multi-region replication at 580K/sec = massive cross-region bandwidth
  - Topic management for 1000 tenants × 50 event types = 50K topics
  - Operational burden of managing 100+ Kafka brokers across 3 regions
  
  SOLUTION: Google Pub/Sub as ingestion layer (serverless, unlimited throughput)
  - Pub/Sub → Dataflow (streaming) for processing
  - Kafka only for internal pub-sub between microservices where ordering matters
  - Pub/Sub handles: global routing, per-tenant topics, automatic scaling

STORAGE:
- Raw events: 5B × 600B = 3 TB/day
- 30-day hot (Bigtable): 90 TB (with TTL auto-deletion)
- 1-year warm (BigQuery): 1.1 PB (queryable, compressed ~300 TB)
- 7-year cold archive (GCS Coldline): 7.7 PB (compressed ~2 PB)
- Enriched events (2x raw): 6 TB/day (stored alongside raw)
- Aggregated metrics: 50 GB/day (time-series rollups per entity per minute)
- Total storage footprint: ~3 PB active, ~10 PB archive
```

### Anomaly Detection Scaling
```
DETECTION PIPELINE COMPUTE (at 58K events/sec):

Layer 1 (Rules — stateless): 
  - 500 rules evaluated per event (tenant-specific + global)
  - CPU cost: ~5μs per event per rule, but batched: ~20μs total per event
  - 58K events/sec × 20μs = 1.16 CPU-seconds per second
  - 2 CPU cores handle this easily per region (TRIVIAL)
  
Layer 2 (Statistical — stateful):
  - Per-entity Z-score, moving averages, seasonal decomposition
  - Entities: 500M globally (1000 tenants × 500K entities avg)
  - State per entity: 500 bytes (multi-metric baselines)
  - Total state: 500M × 500B = 250 GB
  - TOO LARGE for in-memory (Flink RocksDB state max practical: ~50GB/TM)
  - SOLUTION: External state in Bigtable (10ms reads) with local LRU cache
    - Hot entities (1% actively updating): 5M × 500B = 2.5 GB (fits in cache)
    - Cold entities: lazy load from Bigtable on first event
  - Cost: ~200μs per event (cache hit) to ~10ms (Bigtable lookup)
  - Throughput: 58K/sec × 200μs avg = 11.6 CPU-sec/sec = 12 cores
  
Layer 3 (ML — GPU-accelerated):
  - Isolation Forest + LSTM sequence models
  - Only triggered for events that pass L1+L2 (pre-filtered to ~1% = 580/sec)
  - Batch inference: groups of 64 events every 110ms
  - Single A100 GPU: ~10K inferences/sec → handles this easily
  - Sequence model (contextual anomaly): processes entity timelines
    - 500M entities, but only active subset matters: ~5M/day update
    - Batch re-score hourly: 5M × 500ms = 2.5M seconds ÷ 3600 = 700 GPUs
    - OR: trigger only on L2 alerts = 50K/day → single GPU, <1 hour
  
Layer 4 (LLM-based — for novel anomalies, Google-scale addition):
  - For anomalies that don't match known patterns (truly novel)
  - Feed entity context + recent events to Gemini for reasoning
  - Volume: ~500 novel cases/day (1 in 10K anomalies)
  - Cost: $0.05/assessment × 500 = $25/day (trivial at this scale)
  - Value: catches black swan events no rule or statistical model can

TOTAL COMPUTE FOR DETECTION (per region):
  - 50 Dataflow workers (L1+L2), auto-scaling 20-200
  - 4 GPU instances (L3 ML inference)
  - 2 LLM API connections (L4 novel detection)
  - Bigtable state cluster: 50 nodes (250 GB state + read throughput)
  - Cost: ~$200K/month per region × 3 regions = $600K/month

DETECTION FUNNEL (daily, platform-wide):
  5B events → 50M pass L1 (1%) → 5M pass L2 (10% of L1)
  → 500K confirmed anomalies (10% of L2) → 50K actionable alerts (10%)
  → 5K auto-actions triggered → 500 escalations to humans

  Per customer (1000 tenants):
  5M events → 50K L1 → 5K L2 → 500 anomalies → 50 alerts → 5 auto-actions
  THIS IS MANAGEABLE. The filtering cascade is the key design insight.
```

---

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

## CAPACITY PLANNING FOR GROWTH (GOOGLE-SCALE)

```
┌──────────────┬──────────────┬──────────────┬──────────────┬─────────────────────────────┐
│              │ Year 0       │ Year 1       │ Year 2       │ Scaling Action              │
├──────────────┼──────────────┼──────────────┼──────────────┼─────────────────────────────┤
│ Events/day   │ 5B           │ 20B          │ 50B          │ Pub/Sub auto-scales         │
│ Peak events/s│ 580K         │ 2.3M         │ 5.8M         │ Dataflow auto-scaling       │
│ Tenants      │ 200          │ 500          │ 1,000        │ Namespace isolation          │
│ Entities     │ 100M         │ 300M         │ 500M         │ Bigtable-backed state       │
│ Storage/day  │ 3 TB         │ 12 TB        │ 30 TB        │ Tiered (hot→warm→cold)      │
│ Anomalies/day│ 1M           │ 3M           │ 5M           │ ML filtering + correlation  │
│ Alerts/day   │ 10K          │ 30K          │ 50K          │ Intelligent grouping        │
│ Auto-actions │ 1K           │ 5K           │ 15K          │ Confidence-gated expansion  │
│ Pub/Sub tput │ Auto         │ Auto         │ Auto         │ Serverless (unlimited)      │
│ Dataflow wrk │ 100          │ 300          │ 800          │ Auto-scale with backlog     │
│ Bigtable nds │ 50           │ 150          │ 400          │ Scale with entity count     │
│ GPU (ML)     │ 8            │ 20           │ 50           │ Scale with anomaly volume   │
│ Monthly cost │ $600K        │ $1.5M        │ $3.5M        │                             │
│ Cost/tenant  │ $3K          │ $3K          │ $3.5K        │ Sub-linear per-tenant cost! │
│ Revenue/tent │ $15K         │ $20K         │ $25K         │ Expand with value delivered  │
└──────────────┴──────────────┴──────────────┴──────────────┴─────────────────────────────┘

KEY INSIGHTS AT GOOGLE SCALE:

1. Cost scales with EVENTS, not anomalies (same as before, just 1000x bigger)
   - Ingestion + processing: 80% of cost (scales linearly with event volume)
   - Detection (stateful): 15% of cost (scales with entity count, sub-linear)
   - Actions + alerting: 5% of cost (scales with anomaly count — tiny!)

2. Per-tenant cost DECREASES with scale:
   - Shared infrastructure (Pub/Sub, Dataflow, Bigtable) amortized
   - ML models shared across tenants (transfer learning on anomaly patterns)
   - Operational team size grows sub-linearly (automation improves)

3. At Year 2 ($3.5M/month):
   - Ingestion + Dataflow: $2M/month (58% — dominated by compute)
   - Bigtable (state + storage): $800K/month (23%)
   - GPU/ML inference: $400K/month (11%)
   - Alerting + actions: $150K/month (4%)
   - Networking/egress: $150K/month (4%)

4. CRITICAL SCALING INFLECTION POINTS:
   - 1B events/day: Kafka becomes operational burden → migrate to Pub/Sub
   - 100M entities: In-memory state impossible → Bigtable-backed state
   - 10K alerts/day/tenant: Human review impossible → ML severity scoring mandatory
   - 1M auto-actions/day: Need formal verification of action safety
```
