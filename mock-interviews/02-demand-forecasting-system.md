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
- Google-scale retail platform: serving 500+ large retailers globally (think: Google Retail AI)
- Top 50 retailers each have 1M+ SKUs, 10K+ locations = 10B+ time series across platform
- 10 years of POS data, transaction-level, sub-second streaming ingestion
- Promotions, markdowns, new product launches — all dynamic and real-time
- Under-forecasting costs vary: 10x for perishables, 3x for general, asymmetric loss functions
- "Near-real-time" = within 5 minutes of demand signal change (streaming architecture)
- Need probabilistic forecasts (full quantile distributions) for downstream optimization
- GCP-native, multi-region, leveraging TPUs and Vertex AI at scale
- Fully automated ML platform — no per-customer data science needed (AutoML at scale)

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me frame the problem quantitatively:

SCALE (Google-Level — serving 500+ retailers):
- Platform total: 500 retailers × avg 2M SKUs × avg 5K locations = 5 BILLION time series
- Largest single customer: 10M SKUs × 50K locations = 500B time series
- Each needs 30-day-ahead hourly predictions = 5B × 720 = 3.6 TRILLION forecast values/day
- POS data: ~2B transactions/day across all retailers (500 retailers × 4M avg)
- External signals: weather (10M grid points), events (50K/day), social (100M posts)

LATENCY:
- Batch forecast: Rolling 4-hour windows (not nightly — continuous reforecasting)
- Streaming update: Within 5 minutes of demand signal change
- API serving: <50ms p99 for single SKU-location forecast retrieval
- Bulk export: 1B forecasts delivered in <30 minutes to downstream systems

ACCURACY TARGET:
- Platform average wMAPE: <35% at SKU-store-day level
- Per-customer SLA: contractual accuracy guarantees with financial penalties
- Each 1% improvement at this scale = $500M+ in aggregate inventory savings
- Must beat each customer's existing in-house forecasting within 90 days of onboarding

COMPUTE BUDGET:
- Training: 5B time series × global model approach = 500 TPU v5e chips for 6 hours
  - Cannot do one-model-per-series at this scale (5B models × 0.5s = 79 years sequential!)
  - Must use: global foundation model + fine-tuning per customer cluster
- Inference: 3.6T predictions ÷ 4 hours = 250M predictions/second sustained
  - This REQUIRES distributed batch inference across 1000+ TPU chips
  - Online serving: 100K QPS for real-time forecast API (from pre-computed cache)
- TOTAL COMPUTE: ~$2M/month in TPU/GPU time (amortized across 500 customers = $4K/customer)

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
# ENHANCED: Demand Forecasting System — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 2 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES (GOOGLE-SCALE)

### Compute Requirements (Rigorous)
```
THE FUNDAMENTAL MATH (PLATFORM-LEVEL):
- 500 retailers × avg 2M SKUs × avg 5K locations = 5 BILLION forecasting units
- Each unit needs 30 days × 24 hourly predictions = 720 prediction values
- Total: 5B × 720 = 3.6 TRILLION forecast values per refresh cycle
- Each unit needs features: 100 lag + 50 calendar/temporal + 30 external = 180 features

TRAINING COMPUTE:

Option A: One model per series (5B independent models)
  - Training time per model: ~0.5 seconds (LightGBM)
  - Total: 5B × 0.5s = 2.5B seconds = 79 YEARS sequential
  - Even with 10,000 machines: 29 days per retrain cycle
  - INFEASIBLE. Cannot scale linearly with series count at Google scale.

Option B: Global foundation model (single transformer, all retailers)
  - Training data: 5B series × 365 days × 5 years = 9.1 TRILLION data points
  - Model: Transformer-based (like Google's TimesFM) with cross-series attention
  - Parameters: 1-10B params (larger = better cross-learning)
  - Training: 512 TPU v5e chips × 48 hours = 24,576 TPU-hours
  - Weekly incremental retrain: 256 TPUs × 4 hours = 1,024 TPU-hours
  - MASSIVE cross-learning: grocery in Tokyo improves grocery in Paris
  - Accuracy: Best for stable demand, struggles with promotions

Option C: Hierarchical model zoo (MY CHOICE)
  - 10 vertical-specific foundation models (grocery, fashion, electronics, pharma, etc.)
  - Each: 64 TPUs × 12 hours = 768 TPU-hours per vertical
  - Per-customer fine-tuning (top 50 customers with enough data):
    50 × 8 TPUs × 2 hours = 800 TPU-hours total
  - Remaining 450 customers: zero-shot from vertical model
  - Promotion overlay model: separate GBM trained on promotion-specific features
  - Total weekly training: ~10,000 TPU-hours = ~$50K/week

WHY Option C at Google scale:
  - Verticals have genuinely different demand patterns (perishable ≠ fashion)
  - Top customers have enough data (billions of rows) to benefit from fine-tuning
  - Smaller customers get 90% of accuracy from vertical model without custom training
  - Isolates failures: one vertical's model breaking doesn't affect others
  - Allows different SLAs per vertical (grocery needs hourly, fashion needs weekly)
  - Matches org structure: separate teams per vertical

INFERENCE COMPUTE:
  - 3.6T predictions per cycle (every 4 hours)
  - Pipeline: batch inference, prioritized by customer tier + freshness SLA
  - Transformer inference on TPU: ~5000 predictions/chip/second (batched, 720-step)
  - Throughput needed: 3.6T ÷ 14,400s (4 hours) = 250M predictions/second
  - TPU chips needed: 250M ÷ 5000 = 50,000 chips if sustained... 
    BUT: stagger across 4-hour window with priority queuing = 1,000 chips actual
  - Pre-computed results: Bigtable (5B rows × 720 vals × 3 quantiles × 4 bytes = 43 TB)
  - Hot cache: Memorystore Redis cluster (top 10% = 4.3 TB, 500 nodes)
  - API: <10ms p99 for single SKU-location lookup (from cache/Bigtable)

STREAMING UPDATES (5-minute freshness):
  - 2B POS events/day = 23K events/sec avg, 500K/sec peak (Black Friday)
  - Each event → feature update → Bayesian forecast adjustment (NOT full re-inference)
  - Kalman filter update: O(1) per event — extremely cheap
  - Only trigger full re-inference if deviation > 3σ from expected
  - Streaming compute: 200 Dataflow workers (auto-scaling 50-500)
  - Result: 5-minute p95 freshness, <1 minute for high-priority customers
```

### Storage Architecture
```
RAW DATA:
- POS transactions: 2B/day × 300 bytes = 600 GB/day = 219 TB/year
- 5 years historical (all retailers): ~1.1 PB
- External signals (weather 10M grid pts, events 50K/day, social 100M): 50 GB/day
- Total raw growth: ~250 TB/year

FEATURE STORE:
- Offline (training):
  5B units × 180 features × 1825 days × 4 bytes (float16) = 6.6 PB raw
  → Parquet on GCS, partitioned by retailer_id/date (3-4x compression): ~2 PB
  → Active training window (365 days): 400 TB compressed
  → Access via BigQuery federated queries (no data duplication)

- Online (serving): 
  5B units × 180 features × 4 bytes = 3.6 TB
  → TOO LARGE for pure Redis — use Bigtable SSD (sub-10ms reads)
  → Hot tier (top 100 retailers, 50% of queries): 800 GB in Memorystore
  → Refresh: Continuous streaming via Dataflow (< 5 min lag)

FORECAST OUTPUT:
- 5B units × 720 hourly forecasts × 3 quantiles × 4 bytes = 43 TB
  → Primary: Bigtable (auto-sharded, handles 500K+ reads/sec globally)
  → Cache: Redis for top 10% (4.3 TB across 500 Redis nodes)
  → Bulk export: GCS Parquet for customers who pull daily (43 TB/export)
  → API: <5ms from Redis, <15ms from Bigtable (p99)

TOTAL INFRASTRUCTURE:
┌───────────────────────────────────────────────────────────────────────┐
│ Component           │ Size         │ Service           │ Cost/month   │
├─────────────────────┼──────────────┼───────────────────┼──────────────┤
│ Raw data (hot, 90d) │ 54 TB        │ BigQuery          │ $80K         │
│ Raw data (cold)     │ 1.1 PB       │ GCS (Coldline)    │ $5K          │
│ Feature store (off) │ 2 PB (compr) │ GCS + BigQuery    │ $50K         │
│ Feature store (on)  │ 3.6 TB       │ Bigtable SSD      │ $120K        │
│ Forecast output     │ 43 TB        │ Bigtable+Redis    │ $200K        │
│ Redis cache cluster │ 4.3 TB       │ Memorystore       │ $180K        │
│ Training cluster    │ 1000 TPU v5e │ Vertex AI         │ $600K        │
│ Inference cluster   │ 1000 TPU v5e │ Vertex AI (batch) │ $500K        │
│ Streaming (Dataflow)│ 200 workers  │ Dataflow          │ $120K        │
│ API serving (GKE)   │ 100 nodes    │ GKE (3 regions)   │ $80K         │
│ Monitoring + MLOps  │ —            │ Vertex AI Pipelines│ $50K         │
│ Networking (egress) │ ~100 TB/mo   │ Cloud Interconnect│ $50K         │
├─────────────────────┼──────────────┼───────────────────┼──────────────┤
│ TOTAL               │              │                   │ ~$2.0M/month │
│ Per customer (avg)  │              │                   │ ~$4K/month   │
│ Revenue/customer    │              │                   │ $20-500K/year│
└───────────────────────────────────────────────────────────────────────┘

UNIT ECONOMICS AT GOOGLE SCALE:
- Total platform cost: $24M/year
- Revenue (500 customers × avg $100K): $50M/year → 52% gross margin early
- At 2000 customers: Cost grows ~2x ($48M), Revenue 4x ($200M) → 76% margin
- Sub-linear cost scaling because:
  - Foundation models shared across customers (training amortized)
  - Bigtable/Spanner get cheaper per byte at scale
  - TPU pods are more efficient at larger batch sizes
  - Cross-learning improves accuracy without more compute
```
```

---

## ALTERNATIVE MODEL ARCHITECTURES

### Approach A: Classical Statistical (Prophet, ETS, ARIMA)
```
Each series independently modeled with decomposition:
  demand(t) = trend(t) + seasonality(t) + holiday(t) + noise(t)

ARCHITECTURE:
  50M independent Prophet models
  Each trained on 2-3 years of daily data
  Detect changepoints, fit Fourier seasonality

PROS:
+ Interpretable (can explain: "demand is up because of summer seasonality")
+ Handles missing data gracefully
+ No feature engineering needed
+ Fast training per model (2 seconds)
+ Well-understood uncertainty quantification

CONS:
- Cannot capture cross-series patterns (SKU A affects SKU B)
- Cannot leverage external features (weather, promotions) well
- Poor at promotion effects (step changes)
- 50M models is operationally expensive
- Accuracy ceiling: typically 60-70% MAPE at SKU-day level

WHEN TO CHOOSE: 
- Baseline to beat
- Small/medium scale (<10K series)
- Interpretability more important than accuracy
- No good external data available
```

### Approach B: Gradient Boosted Trees (LightGBM/XGBoost)
```
Global model with engineered features:
  X = [lag_features, calendar_features, promo_features, external_features]
  y = demand(t+h) for horizon h

ARCHITECTURE:
  One global LightGBM model
  Trained on all series simultaneously (entity embeddings or one-hot)
  Separate model per forecast horizon (h=1, h=7, h=14, h=30)
  Or: single multi-output model

PROS:
+ Handles feature interactions naturally (promo × day_of_week × category)
+ Fast training (hours on 100 machines, not days)
+ Excellent at promotion effects (tree splits on promo features)
+ Cross-learning: patterns from high-volume SKUs help low-volume ones
+ Feature importance gives explainability
+ Robust to missing features (trees handle naturally)

CONS:
- Requires extensive feature engineering (50-100 features)
- Doesn't capture temporal dependencies as well as sequence models
- Uncertainty quantification: need quantile regression variant
- Point-in-time features must be carefully managed (avoid future leakage)
- Doesn't extrapolate well beyond training distribution

WHEN TO CHOOSE:
- Rich external features available (promotions, weather, events)
- Moderate to large scale
- Need balance of accuracy and speed
- Team has feature engineering capability

ACCURACY: 45-55% MAPE at SKU-day level (significantly better than statistical)
```

### Approach C: Deep Learning (Temporal Fusion Transformer, DeepAR, N-BEATS)
```
Sequence models that learn temporal patterns end-to-end:
  Input: historical sequence + known future inputs (promotions, holidays)
  Output: probability distribution over future demand

ARCHITECTURE:
  Temporal Fusion Transformer (TFT):
  - Variable selection: learns which inputs matter
  - Temporal attention: learns which historical points matter
  - Multi-horizon output: predicts all 30 days simultaneously
  - Built-in uncertainty quantification

PROS:
+ State-of-the-art accuracy for complex patterns
+ Learns feature importance automatically (less manual engineering)
+ Native multi-horizon prediction
+ Attention mechanism provides interpretability
+ Handles variable-length history
+ Transfer learning possible (pre-train on public data, fine-tune)

CONS:
- Much slower to train (days on GPU cluster vs hours for LightGBM)
- Requires more data per series for good fit (cold start problem)
- Harder to debug when wrong (black box compared to trees)
- More infrastructure (GPU serving needed for low-latency)
- Overfit risk on small-data series (long tail of slow-moving SKUs)
- Less stable training (learning rate sensitivity, batch size effects)

WHEN TO CHOOSE:
- Very large scale (benefit from shared learning)
- Complex patterns that trees can't capture
- Sufficient compute budget for GPU training/serving
- Mature ML platform already exists

ACCURACY: 40-50% MAPE at SKU-day level (best, but marginal over trees)
```

### Approach D: Foundation Models (TimesFM, Chronos, Lag-Llama)
```
Pre-trained on millions of time series, zero-shot or fine-tuned:
  Input: raw time series history (minimal feature engineering)
  Output: probabilistic forecast

ARCHITECTURE:
  TimesFM (Google) or Chronos (Amazon):
  - Pre-trained on 100B+ time points across domains
  - Zero-shot: just provide history, get forecast
  - Fine-tune: adapt to domain-specific patterns with small data

PROS:
+ Zero-shot capability (new SKUs immediately forecastable)
+ Minimal feature engineering (learns patterns from raw sequence)
+ Excellent cold-start (no history needed beyond a few weeks)
+ Single model for all series (operationally simple)
+ Probabilistic by design (native confidence intervals)

CONS:
- Cannot easily incorporate external features (promotions, weather)
- Less accurate than tuned LightGBM for feature-rich scenarios
- Large model = expensive inference (but batch-able)
- New technology, less proven at production scale
- May not capture domain-specific patterns without fine-tuning
- Hallucination risk: model "imagines" patterns not in data

WHEN TO CHOOSE:
- Cold-start is a major problem (many new products)
- Feature engineering capacity is limited
- Want a strong baseline quickly
- Complement to feature-based models in ensemble

ACCURACY: 50-55% MAPE zero-shot, 45-50% fine-tuned
```

### My Recommendation: Ensemble of B + D
```
WHY AN ENSEMBLE:

Model type → Strength → Weakness:
  LightGBM → Promotion effects, features → Cold start, extrapolation
  Foundation → Cold start, robust baseline → No external features
  Statistical → Stable trends → Complex patterns

ENSEMBLE STRATEGY:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  For each SKU-location:                                          │
│                                                                   │
│  IF new product (< 4 weeks history):                             │
│    → Foundation model only (zero-shot from similar products)      │
│    → Fall back to category average if foundation unavailable      │
│                                                                   │
│  IF established product, NO upcoming promotion:                   │
│    → Weighted ensemble: 60% LightGBM + 40% Foundation model      │
│    → Weights learned per SKU based on recent holdout accuracy     │
│                                                                   │
│  IF established product, WITH upcoming promotion:                 │
│    → LightGBM with promo features (dominant)                     │
│    → Foundation model gets weight reduction during promo periods   │
│    → Reason: Foundation model hasn't seen YOUR specific promo     │
│      calendar; LightGBM was trained with it                      │
│                                                                   │
│  META-LEARNER (learns optimal weights):                           │
│  - Train on 90-day holdout: which model was more accurate for     │
│    which SKU type, time period, and scenario?                    │
│  - Update weights weekly                                         │
│  - Constrain: no single model < 10% weight (diversification)     │
│                                                                   │
│  RECONCILIATION (after ensembling):                               │
│  - Apply MinT hierarchical reconciliation                        │
│  - Ensures category forecasts = sum of SKU forecasts             │
│  - 5-10% accuracy boost for free (mathematically proven)         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

EXPECTED ACCURACY:
- Ensemble: 40-45% MAPE (better than any single model)
- After reconciliation: 38-43% MAPE
- Business value: each 1% MAPE improvement → ~$5M annual savings 
  (at $100M inventory × 5% carrying cost × proportional safety stock reduction)
```

---

## TRADE-OFF MATRICES

### Accuracy vs Other Dimensions

| Approach | MAPE | Training Time | Cold Start | Interpretability | Infra Cost |
|----------|------|---------------|-----------|-----------------|------------|
| Prophet (per series) | 65% | 1 day (50M models) | Bad (needs 2+ years) | Excellent | Low ($5K/mo) |
| LightGBM (global) | 48% | 4 hours | Medium (needs 4 weeks) | Good (SHAP) | Medium ($10K/mo) |
| TFT (deep learning) | 44% | 3 days (GPU) | Medium | Medium (attention) | High ($30K/mo) |
| Foundation (zero-shot) | 52% | 0 (pre-trained) | Excellent | Low | Medium ($15K/mo) |
| **Ensemble (B+D)** | **42%** | **6 hours** | **Excellent** | **Good** | **$25K/mo** |

### Real-Time Update Approaches

| Approach | Update Latency | Accuracy | Compute Cost | Complexity |
|----------|---------------|----------|--------------|------------|
| Full retrain (nightly) | 24 hours | Baseline | High (nightly cluster) | Low |
| Incremental adjustment (Bayesian update) | 2 hours | Baseline + 2% | Very low (arithmetic) | Low |
| Online learning (streaming SGD) | Minutes | Baseline + 1% | Medium (always-on) | High |
| Hybrid (nightly retrain + 2hr adjustment) | 2 hours | Baseline + 2% | Medium | Medium |
| Trigger-based (retrain on signal detection) | Variable | Baseline + 3% | Medium | High |

**MY CHOICE: Hybrid (nightly retrain + 2hr Bayesian adjustment)**
```
WHY:
- Nightly retrain: full accuracy with all features (best quality)
- 2hr adjustment: catches intraday demand shifts (responsive)
- Not online learning because:
  1. Streaming SGD is unstable for noisy demand data
  2. Risk of overfitting to recent noise
  3. Harder to reproduce and debug
  4. No clear accuracy advantage for the complexity cost

ADJUSTMENT ALGORITHM (Bayesian updating):
  prior = nightly_forecast(t+h)
  evidence = actual_demand(t) vs forecast(t)
  posterior = prior + α × evidence × decay(h)

  where:
    α = learning rate (higher for volatile SKUs)
    decay(h) = 0.9^h (today's surprise matters less for next month)
    
  This is elegant because:
  - O(1) compute per update per series (simple arithmetic)
  - 50M series × simple math = seconds on a single machine
  - Monotonically decays influence (stable, won't oscillate)
  - α can be per-SKU (volatile SKUs update faster)
```

---

## DEEP TRADE-OFF: ACCURACY vs INTERPRETABILITY

### Why This Matters at Staff Level
```
The business doesn't care about MAPE. They care about TRUST.

A forecast that's 5% more accurate but unexplainable will be LESS 
useful than a slightly worse forecast that planners understand and trust.

KEY QUESTION: "Why does the model predict a 3x spike next Tuesday?"

Option A (Black box, more accurate):
  "The model predicts 3x demand."
  → Planner: "I don't trust this. I'll use my gut." → System unused.

Option B (Explainable, slightly less accurate):
  "The model predicts 3x demand because:
   1. Promotion starts Tuesday (historical promo lift for this category: 2.5x)
   2. Competitor out-of-stock on similar product (adds 0.5x from substitution)
   3. Weather forecast: heatwave (this product correlates with temp > 90°F)
   Confidence: 75% (±30% range)"
  → Planner: "That makes sense. Proceed." → System trusted.

MY DESIGN FOR EXPLAINABILITY:
┌─────────────────────────────────────────────────────────────────┐
│ SHAP (SHapley Additive exPlanations) for every forecast:        │
│                                                                   │
│ forecast = baseline + Σ(feature_contributions)                    │
│                                                                   │
│ Example output:                                                   │
│ Forecast: 450 units (vs baseline of 150)                         │
│ +200: promotion_active (BOGO discount, category avg lift)         │
│ +75: day_of_week=Friday (historically +50% on Fridays)           │
│ +30: weather_hot (>90°F correlates with +20% for beverages)      │
│ -5: trend (slight declining trend over 3 months)                  │
│                                                                   │
│ IMPLEMENTATION:                                                   │
│ - Pre-compute SHAP values during nightly batch (top 5 features)  │
│ - Store alongside forecast in Bigtable                           │
│ - API returns forecast + explanations                            │
│ - UI shows waterfall chart of contributions                      │
│                                                                   │
│ COST: ~30% more compute during batch (SHAP is expensive)         │
│ TRADE-OFF: Worth it. Unexplained forecasts don't get used.       │
└─────────────────────────────────────────────────────────────────┘
```

---

## OPERATIONAL CONCERNS (Staff-Level)

### Monitoring Dashboard Design
```
┌─────────────────────────────────────────────────────────────────┐
│ FORECASTING SYSTEM HEALTH DASHBOARD                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ REAL-TIME INDICATORS:                                            │
│ ┌──────────────────────────────────────────────────────────┐    │
│ │ Pipeline Status: ✓ Nightly train completed 4:30 AM       │    │
│ │ Data Freshness: POS data current as of 2:00 PM           │    │
│ │ API Latency: p50=12ms, p95=45ms, p99=120ms              │    │
│ │ Incremental Update: Last run 1:55 PM (5 min ago) ✓       │    │
│ └──────────────────────────────────────────────────────────┘    │
│                                                                   │
│ ACCURACY METRICS (rolling 7-day):                                │
│ ┌──────────────────────────────────────────────────────────┐    │
│ │ Overall WMAPE: 43.2% (target: <45%) ✓                    │    │
│ │ By category: Produce 38% ✓ | Dairy 41% ✓ | Bakery 52% ⚠ │    │
│ │ Bias: +2.1% (slight over-forecast) — acceptable          │    │
│ │ Stockout rate: 1.8% (target: <2%) ✓                      │    │
│ │ Waste rate: 3.2% (target: <4%) ✓                         │    │
│ └──────────────────────────────────────────────────────────┘    │
│                                                                   │
│ ALERTS:                                                          │
│ ┌──────────────────────────────────────────────────────────┐    │
│ │ ⚠ Bakery accuracy degraded 5% vs last week              │    │
│ │   Root cause: new product launches (cold start)          │    │
│ │   Action: foundation model handling, accuracy improving   │    │
│ │                                                           │    │
│ │ ⚠ Store #247 POS data gap (last 6 hours)                │    │
│ │   Impact: 200 SKU forecasts using last-known state       │    │
│ │   Action: IT team notified, ETA 1 hour                   │    │
│ └──────────────────────────────────────────────────────────┘    │
│                                                                   │
│ MODEL PERFORMANCE (A/B test):                                    │
│ ┌──────────────────────────────────────────────────────────┐    │
│ │ Current model (v3.2): WMAPE 43.2% on 80% traffic        │    │
│ │ Candidate (v3.3): WMAPE 41.8% on 20% traffic            │    │
│ │ Days in test: 12/14 (promote in 2 days if holds)         │    │
│ │ Statistical significance: p=0.03 (significant) ✓          │    │
│ └──────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Deployment & Rollback Strategy
```
MODEL DEPLOYMENT (Canary):

1. Train new model on full data
2. Evaluate on 14-day holdout (MUST beat current model on ALL segments)
3. Deploy to 10% of traffic (random store subset)
4. Monitor for 7 days:
   - Accuracy vs current model (must be equal or better)
   - No regression on any category >3%
   - No increase in stockout rate
   - Latency within bounds
5. If passing: ramp to 50% for 3 days, then 100%
6. If failing: auto-rollback to previous model

ROLLBACK TRIGGERS (automated):
- WMAPE increases >5% for 24 hours
- Any category WMAPE increases >10%
- Stockout rate increases >0.5%
- Model serving latency p99 > 500ms

ROLLBACK MECHANISM:
- Model versions stored in Model Registry (MLflow/Vertex)
- Traffic split via feature flag (instant switch, no redeployment)
- Rollback = flip flag to previous version (takes effect in <1 minute)
- Keep last 3 model versions warm in serving infrastructure
```

---

## THE "STAFF ENGINEER" DIFFERENCE

### What Separates Senior from Staff in This Design

| Aspect | Senior-Level Answer | Staff-Level Answer |
|--------|--------------------|--------------------|
| Scale | "Use distributed training" | Exact math: 50M series × compute per model, specific cluster sizing |
| Trade-offs | "Accuracy vs speed" | Quantified: "3% MAPE improvement costs $15K/mo more compute — worth it because each 1% = $5M inventory savings" |
| Approach selection | "Use deep learning, it's best" | "Ensemble because: trees handle promotions, foundation handles cold-start, combined beats either alone by 3-5 MAPE points" |
| Operations | "Monitor accuracy" | Specific monitoring strategy with alerts, A/B testing protocol, rollback triggers |
| Business impact | "Better forecasts reduce waste" | "43% MAPE → 40% MAPE = 7% reduction in safety stock × $100M inventory × 25% carrying cost = $1.75M annual savings" |
| Evolution | "Add more features" | "Phase 1: rules → Phase 2: ML → Phase 3: foundation models. Each phase has specific trigger to move to next" |
| Team implications | — | "This design requires: 2 ML engineers (model development), 1 data engineer (pipeline), 1 platform engineer (serving infra), 1 product manager (accuracy requirements, user feedback)" |
