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
