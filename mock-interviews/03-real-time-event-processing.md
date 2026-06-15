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
