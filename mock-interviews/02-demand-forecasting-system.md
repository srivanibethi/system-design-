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
