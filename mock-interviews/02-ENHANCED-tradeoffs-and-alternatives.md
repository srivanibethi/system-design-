# ENHANCED: Demand Forecasting System — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 2 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES

### Compute Requirements (Rigorous)
```
THE FUNDAMENTAL MATH:
- 100K SKUs × 500 locations = 50M forecasting units
- Each unit needs 30 daily predictions = 1.5B prediction values/day
- Each unit needs features: 50 lag features + 20 calendar + 10 external = 80 features

TRAINING COMPUTE:

Option A: One model per series (50M independent models)
  - Training time per model: ~0.5 seconds (LightGBM on 1000 data points)
  - Total: 50M × 0.5s = 25M seconds = 290 machine-days
  - With 100 machines: 2.9 days
  - Weekly retrain: barely feasible, need large cluster

Option B: Global model (one model, all series as features)
  - Training data: 50M series × 365 days × 3 years = 55B rows
  - Training time: 4-8 hours on GPU cluster (100 GPUs)
  - Much more feasible for weekly/daily retrain
  - Also more accurate (cross-learning between similar series)

Option C: Cluster-based (1000 clusters of similar SKUs, one model per cluster)
  - Training: 1000 models × 30 min each = 500 hours
  - With 50 machines: 10 hours
  - Good balance of specialization and efficiency

MY CHOICE: Option B (global model) + Option C (cluster-based for promotions)
  - Global model captures cross-series patterns efficiently
  - Cluster-based promotion models capture category-specific promo effects

INFERENCE COMPUTE:
  - 50M predictions × 80 features × simple arithmetic = seconds on modern hardware
  - Pre-compute nightly: batch inference on 100 machines = 15 minutes
  - Store in Bigtable: 50M rows × (30 days × 3 quantiles) × 8 bytes = 36 GB
  - Serve from cache: Redis with 36 GB fits in memory easily
```

### Storage Architecture
```
RAW DATA:
- POS transactions: 10M/day × 200 bytes = 2 GB/day = 730 GB/year
- 3 years historical: 2.2 TB
- Growth: 2 TB/year (more stores, more products)

FEATURE STORE:
- Offline (training): 50M units × 80 features × 1095 days × 8 bytes = 44 TB
  → Stored in Parquet on GCS, partitioned by date
  → Only load recent 90 days for training = 4 TB active

- Online (serving): 50M units × 80 features × 8 bytes = 32 GB
  → Fits in Redis cluster (3 nodes × 16 GB)
  → Refresh every 2 hours from stream processor

FORECAST OUTPUT:
- 50M units × 30 days × 3 quantiles × 8 bytes = 36 GB
  → Pre-computed nightly, stored in Bigtable + Redis cache
  → API serves from Redis (sub-ms latency for reads)

TOTAL INFRASTRUCTURE:
┌─────────────────────────────────────────────────┐
│ Component        │ Size    │ Service     │ Cost/mo│
├──────────────────┼─────────┼─────────────┼────────┤
│ Raw data (hot)   │ 730 GB  │ BigQuery    │ $3K    │
│ Raw data (cold)  │ 2.2 TB  │ GCS (cold)  │ $50    │
│ Feature store    │ 4 TB    │ GCS + Redis │ $2K    │
│ Forecast output  │ 36 GB   │ Bigtable    │ $500   │
│ Online features  │ 32 GB   │ Redis       │ $800   │
│ Training cluster │ 100 GPU │ Vertex AI   │ $15K   │
│ Stream processing│ 10 nodes│ Flink/GKE   │ $3K    │
│ Monitoring       │ —       │ Grafana+Prom│ $500   │
├──────────────────┼─────────┼─────────────┼────────┤
│ TOTAL            │         │             │ ~$25K  │
└─────────────────────────────────────────────────┘
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
