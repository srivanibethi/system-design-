# ENHANCED: Disruption Prediction — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 6 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES

### Signal Volume Analysis
```
MULTI-SOURCE SIGNAL INGESTION:

┌─────────────────────────────────────────────────────────────────────┐
│ Source              │ Raw Volume    │ After Filter │ Useful Signals  │
├─────────────────────┼───────────────┼──────────────┼─────────────────┤
│ News (APIs: GDELT,  │ 100K articles │ 5K relevant  │ ~50 events/day  │
│  Reuters, AP)       │ /day          │ to supply ch │                 │
├─────────────────────┼───────────────┼──────────────┼─────────────────┤
│ Weather (NOAA,      │ 50K forecasts │ 2K in supply │ ~20 severe      │
│  ECMWF, local)      │ /day          │ chain regions│ weather/day     │
├─────────────────────┼───────────────┼──────────────┼─────────────────┤
│ Financial (stock,   │ 10K data pts  │ 500 supplier │ ~5 signals/day  │
│  credit, filings)   │ /day          │ specific     │ (bankruptcies,  │
│                     │               │              │  downgrades)     │
├─────────────────────┼───────────────┼──────────────┼─────────────────┤
│ Maritime (AIS ship  │ 5M position   │ 100K on our  │ ~30 congestion/ │
│  tracking)          │ reports/day   │ routes       │  delay signals  │
├─────────────────────┼───────────────┼──────────────┼─────────────────┤
│ Social media        │ 500K posts    │ 1K relevant  │ ~10 signals/day │
│ (Reddit, Twitter)   │ /day          │              │ (strikes, events)│
├─────────────────────┼───────────────┼──────────────┼─────────────────┤
│ TOTAL               │ ~5.7M raw     │ ~109K filtered│ ~115 signals/day│
└─────────────────────────────────────────────────────────────────────┘

SIGNAL FUNNEL:
  5.7M raw data points → 109K potentially relevant → 115 confirmed signals
  → 30 mapped to customer supply chains → 10 actionable alerts/day
  
  REDUCTION RATIO: 570,000:1 (raw to actionable)
  This tells us: THE HARD PROBLEM IS FILTERING, not processing volume.

COMPUTE REQUIREMENTS:
  NLP Pipeline (news/social):
  - 100K articles × classification (100ms each) = 2.8 hours on 1 machine
  - With 4 machines: 42 minutes batch cycle (refresh every hour) ✓
  - Entity extraction: 5K relevant × 500ms = 42 minutes (same machines)
  - Total NLP compute: 4 machines × 16 hours = $200/day

  Weather Processing:
  - 50K forecasts × geospatial intersection with supply chain nodes
  - Compute: trivial (geospatial query, pre-indexed)
  - Cost: negligible

  Maritime Analysis:
  - 5M AIS points → aggregate into port congestion metrics
  - Rolling 24h counts per port, vessel queue length
  - Stream processing: 1 machine handles this easily
  - Cost: ~$100/day

  Financial Monitoring:
  - 10K data points × lookups against customer supplier lists
  - Anomaly detection on financial time series
  - Cost: negligible (small data, simple models)

TOTAL INFRASTRUCTURE:
  - NLP cluster: 4 machines × $0.10/hr = $10K/month
  - Data feeds: news APIs ($3K), weather ($500), maritime ($2K), financial ($1K)
  - Graph DB (Neo4j): $2K/month
  - ML models (risk scoring): 2 GPU machines = $3K/month
  - Storage + serving: $2K/month
  - TOTAL: ~$24K/month
```

---

## ALTERNATIVE APPROACHES TO RISK PREDICTION

### Approach A: Rule-Based Expert System
```
DESIGN:
  Human experts define risk rules:
  - IF news_sentiment(supplier) < -0.5 for 7 days → RISK: Financial
  - IF weather_forecast(region) has hurricane Cat3+ → RISK: Weather
  - IF port_congestion(port) > 90th percentile for 3 days → RISK: Logistics
  - IF credit_rating_downgrade(supplier) → RISK: Financial

  Rules fire → assess exposure → generate alert

PROS:
+ Explainable (each alert points to specific rule + evidence)
+ Fast to implement (weeks, not months)
+ Deterministic (same inputs → same output)
+ No training data needed
+ Domain experts trust it (they wrote the rules)
+ Easy to update (add/modify rules without retraining)

CONS:
- Misses novel patterns (only catches what experts anticipated)
- Threshold tuning is art, not science (what's "severe" enough?)
- Combinatorial explosion (rules for A AND B AND C → exponential)
- Static (doesn't learn from outcomes)
- Missing non-obvious correlations (can't discover what humans don't know)
- False positive management: overly broad rules trigger too often

WHEN TO CHOOSE: V1 of any prediction system. Rules give immediate value
while you collect data for ML approaches. Also: auditable/regulated contexts.

EXPECTED PERFORMANCE:
- Detection rate: ~50% (catches obvious disruptions)
- False positive rate: ~30% (rules are conservative → many false alarms)
- Lead time: Varies by rule (weather: 7 days, financial: 30 days, 
  port congestion: 2-3 days)
```

### Approach B: ML Classification + NLP
```
DESIGN:
  Train models to predict disruption from features:
  - NLP classifier on news: "Is this article about a supply chain disruption?"
  - Financial risk model: "Given supplier metrics, P(distress in 90 days)?"
  - Time-series anomaly: "Is this port congestion abnormal?"
  
  Combine signals → risk score per supply chain node

PROS:
+ Learns from historical disruptions (better calibrated than rules)
+ Handles feature interactions (weather + carrier + time of year → risk)
+ Improves over time as more events occur
+ Can discover patterns humans didn't anticipate
+ Probabilistic output (confidence-aware)

CONS:
- Needs labeled training data (historically labeled disruptions)
- Rare events have too few positive examples for good training
- Model drift: the world changes, past disruptions ≠ future disruptions
- Harder to explain: "The model says risk is high" vs "Rule X triggered"
- Cold start for new customers (no historical disruptions labeled)
- Requires ML infrastructure and expertise to maintain

WHEN TO CHOOSE: After V1 rules have collected 6+ months of labeled data
(true positives, false positives, missed disruptions).

EXPECTED PERFORMANCE:
- Detection rate: ~70% (better recall from learned patterns)
- False positive rate: ~15% (ML better calibrated than static rules)
- Lead time: Similar to rules (depends on signal availability)
```

### Approach C: Knowledge Graph + Causal Reasoning
```
DESIGN:
  Build a knowledge graph of supply chain relationships + causal links:
  - Node: Supplier → Factory → Port → Customer
  - Edges: supplies, routes_through, depends_on
  - Causal: port_closure → shipping_delay → factory_shortage → stockout
  
  When a signal is detected at any node:
  - Traverse causal paths to find affected downstream nodes
  - Estimate impact based on graph structure (centrality, alternatives)

PROS:
+ Captures cascade effects (upstream disruption → downstream impact)
+ Identifies non-obvious vulnerabilities (Tier 2/3 supplier dependencies)
+ Quantifies resilience (alternative paths exist → lower risk)
+ Can answer "what if" questions (remove a node → what happens?)
+ Leverages structure of the problem (supply chains ARE graphs)

CONS:
- Knowledge graph must be built and maintained (expensive)
- Graph completeness is hard (hidden dependencies)
- Causal links are hard to validate (did disruption at A really cause B?)
- Scaling: large enterprises have million-node supply chain graphs
- Keeping graph current: supply chains change constantly

WHEN TO CHOOSE: When you have rich supply chain topology data (from ERP integration)
and the value of cascade prediction justifies the graph maintenance cost.

EXPECTED PERFORMANCE:
- Detection rate: +10% over ML alone (catches cascades)
- Impact estimation: Much better than alternatives (graph propagation)
- Lead time: Enables "early cascade detection" (detect at source before impact)
```

### Approach D: LLM-Powered Intelligence (Emerging)
```
DESIGN:
  Use LLMs as reasoning engines over multi-source intelligence:
  - Feed LLM: news articles + supply chain context + historical patterns
  - Prompt: "Given these signals, assess disruption risk to [supply chain]"
  - LLM synthesizes complex, multi-factor risk assessment
  - Output: structured risk report with reasoning

PROS:
+ Can reason about novel situations (truly unprecedented events)
+ Synthesizes information across sources (news + weather + financial)
+ Natural language explanations (stakeholders understand)
+ Few-shot learning (show examples of past disruptions → generalizes)
+ Handles ambiguity and incomplete information gracefully

CONS:
- Hallucination risk: LLM might "invent" risks that don't exist
- Non-deterministic: different runs → different assessments
- Expensive at scale ($0.05-0.50 per assessment)
- Latency: LLM inference takes seconds (not real-time)
- Hard to validate: how do you test a reasoning engine?
- Prompt sensitivity: small prompt changes → very different outputs

WHEN TO CHOOSE: As an augmentation layer (not primary). Use LLM to:
- Synthesize multi-source signals into human-readable briefings
- Assess unprecedented event types that ML models haven't seen
- Generate "what if" scenario analyses
- NOT for automated high-frequency alerting (too expensive, too variable)

EXPECTED PERFORMANCE:
- Quality of analysis: Excellent for novel scenarios
- Consistency: Poor (need temperature=0 and deterministic prompting)
- Cost-effective only at low volumes (<100 assessments/day)
```

### My Architecture Decision: LAYERED APPROACH
```
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 1: Rules (immediate, cheap, explainable)                   │
│   - Catches: obvious disruptions with known signatures           │
│   - Coverage: 50% of detectable disruptions                     │
│   - Latency: seconds                                             │
│                                                                   │
│ LAYER 2: ML Models (hours, moderate cost, learned patterns)      │
│   - Catches: subtle patterns, feature interactions               │
│   - Coverage: additional 20% (total: 70%)                       │
│   - Latency: minutes (batch scoring every hour)                  │
│                                                                   │
│ LAYER 3: Knowledge Graph (cascade propagation)                   │
│   - Catches: downstream impacts, hidden dependencies             │
│   - Coverage: additional 10% from cascade detection (total: 80%) │
│   - Latency: seconds (graph traversal is fast once signal detected) │
│                                                                   │
│ LAYER 4: LLM Synthesis (for high-severity alerts)                │
│   - NOT for detection, but for ASSESSMENT and EXPLANATION        │
│   - Takes signals from L1-L3 → generates human-readable briefing │
│   - Only triggered on HIGH severity (saves cost)                 │
│   - Output: "Here's what's happening, here's the impact, here   │
│     are recommended actions with reasoning"                      │
│                                                                   │
│ SYNERGY:                                                         │
│   L1 provides signal → L3 propagates through graph →             │
│   L2 validates (is this real or noise?) → L4 explains to human   │
└─────────────────────────────────────────────────────────────────┘
```

---

## TRADE-OFF MATRICES

### Data Source Cost-Benefit Analysis
```
┌────────────────────┬──────────┬──────────────┬──────────────┬──────────────┐
│ Data Source        │ Cost/mo  │ Detection    │ Lead Time    │ ROI Score    │
│                    │          │ Contribution │              │ (value/cost) │
├────────────────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ News APIs (GDELT,  │ $3K      │ 30%          │ Hours-days   │ HIGH         │
│  Reuters)          │          │              │              │ (broad, cheap)│
├────────────────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ Weather (NOAA+     │ $500     │ 25%          │ 3-14 days    │ VERY HIGH    │
│  commercial)       │          │              │              │ (best ROI)   │
├────────────────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ Maritime AIS       │ $2K      │ 20%          │ 2-5 days     │ HIGH         │
│                    │          │              │              │ (unique signal)│
├────────────────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ Financial data     │ $1K      │ 15%          │ 30-90 days   │ MEDIUM       │
│ (stock, credit)    │          │              │              │ (long lead)  │
├────────────────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ Social media       │ $500     │ 5%           │ Hours        │ LOW          │
│                    │          │              │              │ (noisy)      │
├────────────────────┼──────────┼──────────────┼──────────────┼──────────────┤
│ Satellite imagery  │ $10K     │ 5%           │ Hours-days   │ LOW          │
│                    │          │              │              │ (expensive)  │
└────────────────────┴──────────┴──────────────┴──────────────┴──────────────┘

PRIORITIZATION (build in order of ROI):
1. Weather (highest ROI, most predictable, cheapest)
2. News (broad coverage, moderate cost)
3. Maritime AIS (unique signal for logistics-heavy companies)
4. Financial (longer lead time, supplements news)
5. Social media (marginal value, high noise)
6. Satellite (expensive, niche use cases only)
```

### False Positive vs False Negative Trade-off
```
THE FUNDAMENTAL TENSION:
  More sensitive → catch more real disruptions BUT more false alarms
  Less sensitive → fewer false alarms BUT miss real disruptions

COST ANALYSIS:
  Cost of False Positive (FP):
  - Planner investigates: 30 minutes × $75/hour = $37.50
  - At 5 FP/day: $187.50/day = $5,600/month
  - WORSE: Alert fatigue → planners start ignoring ALL alerts
  
  Cost of False Negative (FN):
  - Missed disruption: stockout, SLA penalty, expediting costs
  - Average missed disruption cost: $50K-$500K
  - At 1 missed disruption/month: $50K-$500K/month
  
  RATIO: FN cost is 10-100x FP cost
  → BIAS TOWARD SENSITIVITY (err on side of alerting)
  → BUT: must manage alert fatigue (aggregate, severity-tier)

OPERATING POINT SELECTION:
┌──────────────────────────────────────────────────────────────────┐
│ Sensitivity │ FP Rate │ Alerts/day │ Missed/month │ Net Cost/mo │
├─────────────┼─────────┼────────────┼──────────────┼─────────────┤
│ 95%         │ 40%     │ 25 alerts  │ 0.5          │ $30K (FN)   │
│ (aggressive)│         │ (10 real)  │              │ + $15K (FP) │
│             │         │            │              │ = $45K      │
├─────────────┼─────────┼────────────┼──────────────┼─────────────┤
│ 80%         │ 20%     │ 12 alerts  │ 2            │ $120K (FN)  │
│ (balanced)  │         │ (10 real)  │              │ + $4K (FP)  │
│             │         │            │              │ = $124K     │
├─────────────┼─────────┼────────────┼──────────────┼─────────────┤
│ 60%         │ 5%      │ 7 alerts   │ 4            │ $240K (FN)  │
│(conservative│         │ (6 real)   │              │ + $1K (FP)  │
│             │         │            │              │ = $241K     │
└──────────────────────────────────────────────────────────────────┘

OPTIMAL: 95% sensitivity with smart alert management
- Yes, 40% FP rate sounds high
- BUT: severity tiering means only HIGH alerts page humans
- Low-severity FPs go to digest → minimal disruption
- The $15K/month FP cost is worth catching 0.5 more disruptions ($30K each)
```

### Prediction Horizon vs Accuracy Trade-off
```
KEY INSIGHT: Prediction gets MUCH harder with longer horizons

┌───────────────────────────────────────────────────────────────────────┐
│ Horizon    │ Accuracy │ Actionable?                │ Use Case          │
├────────────┼──────────┼────────────────────────────┼───────────────────┤
│ 0-24 hours │ 90%      │ Emergency response only    │ "It's happening"  │
│            │          │ (reroute, expedite)         │ Fast alert        │
├────────────┼──────────┼────────────────────────────┼───────────────────┤
│ 1-7 days   │ 70%      │ Tactical (reroute, pre-    │ "Likely soon"     │
│            │          │ order from alt supplier)    │ Prepare contingency│
├────────────┼──────────┼────────────────────────────┼───────────────────┤
│ 1-4 weeks  │ 50%      │ Strategic (build buffer,   │ "Elevated risk"   │
│            │          │ qualify alt suppliers)      │ Increase readiness│
├────────────┼──────────┼────────────────────────────┼───────────────────┤
│ 1-3 months │ 30%      │ Planning (diversify supply │ "Watch this space"│
│            │          │ chain, hedge)               │ Strategic planning│
├────────────┼──────────┼────────────────────────────┼───────────────────┤
│ 3+ months  │ 10%      │ Not reliable enough for    │ Scenario planning │
│            │          │ specific actions            │ only              │
└───────────────────────────────────────────────────────────────────────┘

DESIGN IMPLICATION:
- Don't present all horizons equally (user loses trust if 3-month 
  predictions are wrong)
- Label: "High confidence: port congestion likely this week (85%)"
  vs "Watch: potential sanctions in 2-3 months (35%)"
- Different actions for different horizons:
  Short (tactical): automated recommendations
  Long (strategic): presented in weekly risk briefing, not alerts
```

---

## KNOWLEDGE GRAPH DESIGN FOR SUPPLY CHAIN

```
GRAPH SCHEMA:

NODES:
┌─────────────────────────────────────────────────────────────────┐
│ Type        │ Properties                   │ Example             │
├─────────────┼──────────────────────────────┼─────────────────────┤
│ Supplier    │ name, location, financial_   │ "Foxconn, Shenzhen, │
│             │ health, reliability_score    │  health=72, rel=0.91"│
├─────────────┼──────────────────────────────┼─────────────────────┤
│ Facility    │ type, location, capacity,    │ "Factory, Guangzhou, │
│             │ criticality                  │  10K units/day"      │
├─────────────┼──────────────────────────────┼─────────────────────┤
│ Port        │ location, avg_dwell_time,    │ "Shanghai, 3.2 days, │
│             │ current_congestion           │  congestion=0.78"    │
├─────────────┼──────────────────────────────┼─────────────────────┤
│ Product     │ sku, category, demand_vol,   │ "SKU-789, Electronics│
│             │ margin, substitutes          │  100K/month, $45"    │
├─────────────┼──────────────────────────────┼─────────────────────┤
│ Route       │ origin, destination, mode,   │ "SH→LA, ocean, 14d, │
│             │ typical_time, reliability    │  reliability=0.92"   │
└─────────────────────────────────────────────────────────────────┘

EDGES (relationships):
  (Supplier)-[:SUPPLIES {volume, lead_time}]->(Product)
  (Supplier)-[:LOCATED_IN]->(Region)
  (Product)-[:MANUFACTURED_AT]->(Facility)
  (Facility)-[:ROUTES_THROUGH]->(Port)
  (Product)-[:SUBSTITUTE_FOR]->(Product)
  (Supplier)-[:ALTERNATIVE_FOR]->(Supplier)
  (Route)-[:DEPENDS_ON]->(Port)

VULNERABILITY QUERIES:
  "Single-source materials":
  MATCH (p:Product) WHERE size((p)<-[:SUPPLIES]-()) = 1
  RETURN p.name, p.demand_volume -- These are your highest risk items

  "Cascade impact of port closure":
  MATCH (port:Port {name: 'Shanghai'})<-[:ROUTES_THROUGH]-(f:Facility)
        <-[:MANUFACTURED_AT]-(p:Product)
  RETURN p.name, p.demand_volume, p.margin
  ORDER BY p.margin * p.demand_volume DESC -- Highest $ at risk first

  "Supplier concentration risk":
  MATCH (s:Supplier)-[:SUPPLIES]->(p:Product)
  WITH s, count(p) as product_count, sum(p.margin * p.demand_volume) as total_value
  WHERE total_value > 1000000
  RETURN s.name, product_count, total_value -- Suppliers we over-depend on
```

---

## SYSTEM EVOLUTION & BUILD SEQUENCE

```
MONTH 1-2: "Newsroom" (Pure Detection)
  Build: News monitoring + entity matching + basic alert
  Value: Know about disruptions faster than checking news manually
  Investment: 2 engineers × 2 months = $80K
  Metric: "Detection time vs human baseline" (hours → minutes)

MONTH 3-4: "Weather Shield" (First Prediction)
  Build: Weather forecasting integration + supply chain overlay
  Value: 3-7 day advance warning for weather disruptions
  Investment: +1 engineer × 2 months = $40K
  Metric: "Advance warning time" and "false positive rate"

MONTH 5-7: "Risk Graph" (Cascade Analysis)
  Build: Supply chain knowledge graph + exposure mapping
  Value: Instant "who's affected?" for any signal
  Investment: 2 engineers × 3 months = $120K
  Metric: "Exposure identification time" (hours → seconds)

MONTH 8-10: "Predictive Risk" (ML Models)
  Build: Supplier financial risk model + port congestion prediction
  Value: 30-90 day advance warning for financial risks
  Investment: 1 ML engineer × 3 months = $60K
  Metric: "Prediction accuracy" and "lead time"

MONTH 11-12: "Recommendation Engine" (Actionable)
  Build: Automated recommendation generation (pre-order, reroute, etc)
  Value: Not just "here's the risk" but "here's what to do about it"
  Investment: 2 engineers × 2 months = $80K
  Metric: "Recommendation acceptance rate" and "$ saved per recommendation"

TOTAL YEAR 1: ~$380K investment
EXPECTED SAVINGS: $2-5M/year per large enterprise customer
  (Based on: 10 disruptions/year × $200K-500K avg impact × 50% mitigation)
```
