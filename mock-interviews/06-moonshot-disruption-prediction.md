# Mock Interview 6: Supply Chain Disruption Prediction System (Moonshot)

## Interview Format: 45-minute System Design (Staff Level) — Novel Problem

---

## The Prompt (What the Interviewer Says)

> "Design a system that predicts supply chain disruptions before they happen — anywhere from hours to weeks in advance. This could be anything: a factory fire, a port closure, a pandemic, a geopolitical event, a supplier bankruptcy, extreme weather. The system should assess risk, predict impact, and recommend preemptive actions. This is a hard, open-ended problem — think big."

---

## Why This is a "Moonshot" Question

This question has no standard answer. The interviewer is evaluating:
1. How you decompose an impossibly broad problem into tractable pieces
2. How you handle deep uncertainty (many disruptions are unprecedented)
3. Whether you can think in first principles vs just reciting patterns
4. Your ability to propose novel approaches while being honest about limitations

---

## PHASE 1: Clarifying Questions & Problem Decomposition (8 minutes)

### Questions YOU Should Ask:

**Scoping the Problem:**
1. "Let me decompose 'supply chain disruptions' — I see at least 5 categories with very different prediction approaches. Should I design for all, or focus on a subset?"
   ```
   Category 1: Weather (hurricanes, floods, heat waves)
     → Predictability: 3-14 days ahead, high confidence
     
   Category 2: Geopolitical (sanctions, tariffs, war)
     → Predictability: signals weeks ahead, high uncertainty
     
   Category 3: Supplier-specific (bankruptcy, quality failure, fire)
     → Predictability: some financial signals, mostly surprise
     
   Category 4: Infrastructure (port congestion, transport strike)
     → Predictability: days to weeks, depends on root cause
     
   Category 5: Demand shock (viral trend, competitor exit, pandemic)
     → Predictability: hours to days after onset, hard to predict onset
   ```

2. "Who is the customer for this system? A single company protecting its supply chain, or a platform serving many companies?"
3. "What's the prediction horizon that matters most — 24 hours? 1 week? 1 month?"
4. "What constitutes a 'disruption'? Any delay? Or only significant impacts (>$X damage)?"
5. "How do we validate predictions? Some disruptions are prevented (survivorship bias), and some are so rare we can't train on them."

**About Capabilities:**
6. "Can we acquire external data? (News APIs, satellite imagery, financial data, social media, weather)"
7. "Do we have access to the company's supply chain graph? (Who supplies what, through which routes)"
8. "What's the tolerance for false positives? Alert fatigue is a real risk."
9. "Budget for data acquisition? (Satellite feeds, premium news, financial data = expensive)"

### Interviewer's Likely Answer (Google-Scale):
- All categories in scope — build the "weather service" for global supply chains
- Platform serving 5,000+ enterprises (Google-scale multi-tenant, cross-customer intelligence)
- Prediction horizons: 1 hour (operational), 1-3 days (tactical), 1-4 weeks (strategic)
- Significant disruptions: >$1M potential impact OR affecting 100+ entities
- External data: unlimited budget — satellite, AIS, financial, news, social, government
- False positive tolerance: <10% (enterprise customers have zero patience for noise)
- Supply chain graph: 10B+ nodes across all customers (shared graph where entities overlap)
- Cross-customer intelligence: if Supplier X is failing Customer A, warn Customer B too
- Speed: detection within 5 minutes of earliest observable signal

---

## PHASE 2: First Principles Framing (5 minutes)

```
"Let me frame this from first principles. 

A supply chain disruption has three components:
1. A HAZARD (something bad happens somewhere in the world)
2. An EXPOSURE (a supply chain element is in the blast radius)
3. An IMPACT (the disruption propagates through the supply chain)

My system needs three corresponding capabilities:
1. HAZARD DETECTION: Detect or predict the bad event
2. EXPOSURE MAPPING: Know which supply chain elements are affected
3. IMPACT PROPAGATION: Model how disruption cascades through the network

Prediction difficulty varies by category:

                    Predictable ←────────→ Unpredictable
                    │                              │
   Weather ─────────┤                              │
   Infrastructure ──┤                              │
   Geopolitical ────────────────┤                  │
   Supplier failure ─────────────────────┤         │
   Black swans ────────────────────────────────────┤

For predictable categories: BUILD PREDICTION MODELS
For unpredictable categories: BUILD EARLY DETECTION + RAPID RESPONSE

This is NOT just an ML problem. It's a multi-source intelligence 
system with human-in-the-loop judgment."
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    SIGNAL COLLECTION LAYER                               │
│                                                                          │
│  ┌───────────┐ ┌────────────┐ ┌───────────┐ ┌─────────┐ ┌──────────┐│
│  │ News &    │ │ Weather &  │ │ Financial │ │ Social  │ │ Satellite││
│  │ Media     │ │ Climate    │ │ Markets   │ │ Media   │ │ & IoT    ││
│  │ (100K     │ │ (forecasts,│ │ (supplier │ │ (Twitter│ │ (port    ││
│  │  articles/│ │  satellite,│ │  stock,   │ │  Reddit,│ │  imagery,││
│  │  day)     │ │  sensors)  │ │  credit   │ │  forums)│ │  AIS ship││
│  │           │ │            │ │  ratings) │ │         │ │  tracking││
│  └─────┬─────┘ └─────┬──────┘ └─────┬─────┘ └────┬────┘ └─────┬────┘│
└────────┼──────────────┼──────────────┼────────────┼────────────┼─────┘
         │              │              │            │            │
         ▼              ▼              ▼            ▼            ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  SIGNAL PROCESSING LAYER                                 │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ NLP PIPELINE (for text sources)                                  │    │
│  │ - Entity extraction: companies, locations, events                │    │
│  │ - Event detection: "fire at factory", "strike announced"         │    │
│  │ - Sentiment: escalation detection (getting worse over time?)     │    │
│  │ - Geo-coding: map events to locations                            │    │
│  │ - Deduplication: same event from multiple sources                │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ ANOMALY DETECTION (for numeric/spatial sources)                  │    │
│  │ - Weather: severe weather alerts exceeding thresholds            │    │
│  │ - Financial: credit score drops, stock price crashes             │    │
│  │ - Satellite: port congestion (vessel count above normal)         │    │
│  │ - IoT: unusual patterns at facilities                            │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  OUTPUT: Normalized SIGNAL objects                                       │
│  {signal_id, type, severity, location, affected_entities,               │
│   confidence, source, timestamp, predicted_duration}                     │
└─────────────────────────────────────────────────────┬────────────────────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                KNOWLEDGE GRAPH & EXPOSURE MAPPING                        │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ SUPPLY CHAIN KNOWLEDGE GRAPH (per customer)                     │    │
│  │                                                                  │    │
│  │ Nodes:                                                           │    │
│  │   [Suppliers] → [Materials] → [Factories] → [Products]          │    │
│  │   [Ports] → [Routes] → [Warehouses] → [Customers]               │    │
│  │                                                                  │    │
│  │ Node attributes:                                                 │    │
│  │   - Geographic coordinates (for spatial hazard matching)         │    │
│  │   - Criticality score (single-source? alternative exists?)       │    │
│  │   - Historical reliability                                       │    │
│  │   - Financial health indicators                                  │    │
│  │                                                                  │    │
│  │ Edges:                                                           │    │
│  │   - "supplies" (with volume, lead time, contract terms)          │    │
│  │   - "routes through" (mode, typical transit time)                │    │
│  │   - "alternative to" (substitution possibility)                  │    │
│  │                                                                  │    │
│  │ EXPOSURE QUERY:                                                  │    │
│  │   Signal: "Port of Shanghai congestion"                          │    │
│  │   Query: "Which of my supply chains route through Shanghai?"     │    │
│  │   Result: 47 active shipments, 12 suppliers, affecting 200 SKUs  │    │
│  │   Criticality: 3 are single-source materials (HIGH RISK)         │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┬────────────────────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                RISK ASSESSMENT & PREDICTION ENGINE                       │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ RISK SCORING MODEL                                               │    │
│  │                                                                  │    │
│  │ Risk Score = P(disruption) × Severity × Exposure × (1/Resilience)│   │
│  │                                                                  │    │
│  │ P(disruption): probability event causes actual supply disruption │    │
│  │   - Derived from: signal severity, historical base rates,        │    │
│  │     geographic proximity, duration prediction                    │    │
│  │                                                                  │    │
│  │ Severity: if disrupted, how bad is it?                           │    │
│  │   - Duration estimate (1 day vs 1 month)                         │    │
│  │   - Capacity reduction (10% slowdown vs 100% shutdown)           │    │
│  │                                                                  │    │
│  │ Exposure: how much of my supply chain is in the blast radius?    │    │
│  │   - Number of affected nodes                                     │    │
│  │   - Dollar value at risk                                         │    │
│  │   - SKU criticality (stockout impact)                            │    │
│  │                                                                  │    │
│  │ Resilience: how well can we absorb this?                         │    │
│  │   - Alternative suppliers available?                              │    │
│  │   - Safety stock coverage?                                       │    │
│  │   - Alternative routes exist?                                    │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ PREDICTION MODELS (per disruption category)                      │    │
│  │                                                                  │    │
│  │ Weather Prediction:                                              │    │
│  │   Input: NOAA forecasts, satellite data, historical patterns     │    │
│  │   Output: P(severe weather at location X in next N days)         │    │
│  │   Method: Ensemble of meteorological models + our logistics impact│   │
│  │                                                                  │    │
│  │ Supplier Financial Health:                                       │    │
│  │   Input: Stock price, credit ratings, payment behavior,          │    │
│  │          job postings, news sentiment, Glassdoor reviews         │    │
│  │   Output: P(supplier distress in next 90 days)                   │    │
│  │   Method: Gradient boosted classifier on financial indicators    │    │
│  │                                                                  │    │
│  │ Geopolitical Risk:                                               │    │
│  │   Input: News volume/sentiment, diplomatic events, sanctions     │    │
│  │   Output: Risk score per country/region (relative, not absolute) │    │
│  │   Method: NLP + time-series on news features (HARD, low accuracy)│    │
│  │                                                                  │    │
│  │ Infrastructure:                                                  │    │
│  │   Input: Port AIS data, congestion history, labor news           │    │
│  │   Output: P(port delay > 3 days in next week)                    │    │
│  │   Method: Time-series forecasting on congestion metrics          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┬────────────────────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                  IMPACT SIMULATION & RECOMMENDATION                      │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ PROPAGATION MODEL                                                │    │
│  │                                                                  │    │
│  │ Given: disruption at node X with severity S and duration D       │    │
│  │ Simulate: cascade through supply chain graph                     │    │
│  │                                                                  │    │
│  │ Example:                                                         │    │
│  │   "Supplier A shut down for 2 weeks"                             │    │
│  │     → Material M unavailable                                     │    │
│  │       → Product P cannot be manufactured after day 5 (buffer runs out) │
│  │         → 3 customers affected, $2.3M revenue at risk            │    │
│  │         → Alternative supplier B can supply in 10 days (partial) │    │
│  │                                                                  │    │
│  │ Method: Discrete event simulation on supply chain graph          │    │
│  │ - Consume existing inventory buffers first                       │    │
│  │ - Model production dependencies (BOM explosion)                  │    │
│  │ - Track which end customers are affected and when                │    │
│  │ - Monte Carlo: simulate with uncertainty in duration             │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ RECOMMENDATION ENGINE                                            │    │
│  │                                                                  │    │
│  │ Possible actions (ordered by cost/reversibility):                │    │
│  │ 1. MONITOR: Do nothing yet, watch for escalation                 │    │
│  │ 2. ALERT: Notify stakeholders, prepare contingency               │    │
│  │ 3. PRE-ORDER: Place safety orders with alternative suppliers     │    │
│  │ 4. REROUTE: Redirect in-transit shipments                        │    │
│  │ 5. EXPEDITE: Pay for faster shipping on critical items           │    │
│  │ 6. SUBSTITUTE: Use alternative product/material                  │    │
│  │ 7. COMMUNICATE: Proactively inform affected customers            │    │
│  │                                                                  │    │
│  │ Decision logic:                                                  │    │
│  │ if risk_score > CRITICAL_THRESHOLD and action_cost < impact_cost:│    │
│  │     recommend_action(preemptive=True)                            │    │
│  │ elif risk_score > HIGH_THRESHOLD:                                │    │
│  │     recommend_action(prepare_contingency=True)                   │    │
│  │ else:                                                            │    │
│  │     monitor_and_reassess(frequency=6_hours)                      │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Multi-Source Signal Fusion

```
"The core challenge: How do you combine signals from news, weather, 
financial data, and satellite imagery into a coherent risk assessment?

THE SIGNAL FUSION PROBLEM:
- Same event appears in multiple sources with different details
- Different sources have different latencies (Twitter fast, news slow, 
  financial data very slow)
- Different sources have different reliability
- False signals are common (news is sensationalized, social media is noisy)

MY APPROACH: Event-Centric Fusion

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Step 1: EVENT CANDIDATE GENERATION                               │
│  - Each source independently generates 'event candidates'        │
│  - A candidate = {type, location, severity, confidence, source}  │
│                                                                   │
│  Step 2: EVENT CORRELATION                                        │
│  - Cluster candidates by: location proximity + time window + type │
│  - "Factory fire in Shenzhen" from news + "thermal anomaly in    │
│    Shenzhen industrial zone" from satellite = SAME EVENT          │
│  - Use entity resolution: "Foxconn" + "Hon Hai" = same company   │
│                                                                   │
│  Step 3: CONFIDENCE SCORING                                       │
│  - Single-source event: confidence = source_reliability × 0.5    │
│  - Multi-source corroborated: confidence boosted proportionally   │
│  - Contradicting sources: flag for investigation                  │
│                                                                   │
│  Confidence calculation:                                          │
│  P(real event) = 1 - Π(1 - P(source_i_correct))                 │
│                                                                   │
│  Example:                                                         │
│  - News source (70% reliable) reports port congestion             │
│  - Satellite confirms (85% reliable) high vessel density          │
│  - P(real) = 1 - (0.3 × 0.15) = 1 - 0.045 = 95.5%              │
│                                                                   │
│  Step 4: SEVERITY ESTIMATION                                      │
│  - Use historical analogies: "Similar events in the past caused   │
│    X days of disruption with Y% probability"                     │
│  - Adjust for specifics: scale, affected entity, context          │
│  - Output: probability distribution over severity levels          │
│                                                                   │
│  Step 5: TEMPORAL TRACKING                                        │
│  - Events evolve over time (strike announced → negotiations →     │
│    strike begins → resolution)                                   │
│  - Track event lifecycle with state machine:                      │
│    DETECTED → CONFIRMED → ESCALATING → PEAK → RESOLVING → RESOLVED │
│  - Update risk score as event progresses                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

HANDLING NOISE (the hardest part):
- News mentions 'supply chain' 50K times/day. 99.9% is noise.
- Multi-stage filtering:
  1. Relevance filter (LLM classifier): "Is this about a specific 
     disruption, or just commentary?" → drops 95%
  2. Entity filter: "Does this mention an entity in our customer's 
     supply chain?" → drops 80% of remainder
  3. Severity filter: "Is this significant enough to matter?" → drops 50%
  4. Result: ~50 actionable signals per day from 50K raw mentions
"
```

---

### Deep Dive 2: Handling Prediction for Rare Events

```
"The fundamental challenge: truly disruptive events are RARE. 
How do you train models for events that happen once every 10 years?

THE RARITY PROBLEM:
- Pandemics: ~3 in 100 years
- Port closures: ~5-10/year globally
- Supplier bankruptcies: varies, but specific ones are unpredictable
- We can't train on what hasn't happened

MY APPROACH: Multi-Strategy for Different Predictability Levels

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  STRATEGY 1: ANALOG-BASED REASONING (for unprecedented events)   │
│  ──────────────────────────────────────────────────────────────  │
│  Can't predict Black Swans, but can reason about impact:          │
│                                                                   │
│  "We've never seen a pandemic close Chinese factories before,     │
│   but we CAN model: 'If Chinese factories are closed for X weeks, │
│   what happens to our supply chain?'"                             │
│                                                                   │
│  → SCENARIO-BASED: Pre-compute impact for hypothetical disruptions│
│    - "What if Supplier A goes down for 2 weeks?"                  │
│    - "What if Port X closes for 1 month?"                         │
│    - Generate a library of impact scenarios                       │
│    - When real event occurs → match to closest scenario           │
│                                                                   │
│  STRATEGY 2: LEADING INDICATORS (for predictable categories)      │
│  ──────────────────────────────────────────────────────────────  │
│  Some disruptions have reliable precursors:                       │
│                                                                   │
│  Supplier bankruptcy (30-90 day warning signs):                   │
│    - Stock price decline > 30% in 90 days                        │
│    - Credit rating downgrade                                      │
│    - Payment behavior change (paying invoices late)               │
│    - Layoff announcements / hiring freeze                         │
│    - Key executive departures                                     │
│    - Negative news sentiment acceleration                         │
│                                                                   │
│  Port congestion (3-7 day warning):                              │
│    - Vessel queue length increasing                               │
│    - Average wait time trending up                                │
│    - Nearby ports seeing overflow traffic                         │
│    - Labor union rhetoric escalating                              │
│                                                                   │
│  STRATEGY 3: TRANSFER LEARNING FROM ADJACENT DOMAINS              │
│  ──────────────────────────────────────────────────────────────  │
│  - Insurance industry has extensive catastrophe models            │
│  - Climate science has weather prediction models                  │
│  - Epidemiology has disease spread models                         │
│  - Financial markets are leading indicators for economic shocks   │
│  → Integrate these external models as features                    │
│                                                                   │
│  STRATEGY 4: HUMAN INTELLIGENCE AUGMENTATION                     │
│  ──────────────────────────────────────────────────────────────  │
│  For truly novel events, AI alone is insufficient:               │
│  - Industry analysts provide qualitative risk assessments         │
│  - On-ground contacts at key locations provide local intel        │
│  - "Prediction markets" among supply chain professionals          │
│  - LLM-powered synthesis: "Given these 10 signals, what's        │
│    your assessment?" → structured by AI, judged by human          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

HONEST LIMITATIONS (say this in the interview):
"Let me be explicit about what this system CANNOT do:
1. Predict Black Swans (by definition, unprecedented events are unpredictable)
2. Assign precise probabilities to rare events (not enough training data)
3. Replace human judgment for geopolitical risk (too complex)

What it CAN do:
1. Detect disruptions faster than humans monitoring manually
2. Instantly assess exposure and impact (graph traversal)
3. Quantify the blast radius before it cascades
4. Recommend preemptive actions that reduce impact
5. Learn from each disruption to detect similar patterns next time"
```

---

### Deep Dive 3: Evaluation — How Do You Know This System Works?

```
"This is a critical question for any prediction system, especially one 
predicting rare events.

THE EVALUATION CHALLENGE:
- Can't wait for rare events to measure accuracy (too slow)
- Prevented disruptions are invisible (survivorship bias)
- False positives are costly (erode trust) but false negatives are 
  catastrophic (miss real disruptions)

MY EVALUATION FRAMEWORK:

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  LEVEL 1: COMPONENT EVALUATION (continuous)                      │
│  - Signal detection precision/recall per source                  │
│  - Entity resolution accuracy (are we matching events correctly?)│
│  - Severity estimation calibration (are our probabilities real?) │
│  - Exposure mapping coverage (do we know about all supply paths?)│
│                                                                   │
│  LEVEL 2: BACKTESTING (monthly)                                  │
│  - Replay historical disruptions through the system              │
│  - "Given data available BEFORE event X, would we have detected  │
│    it, and how early?"                                           │
│  - Use: COVID factory closures (Jan 2020), Suez Canal (Mar 2021),│
│    Texas freeze (Feb 2021), Shanghai lockdown (Apr 2022)         │
│  - Measure: detection lead time, severity accuracy, action quality│
│                                                                   │
│  LEVEL 3: SIMULATED DISRUPTIONS (quarterly)                      │
│  - Inject synthetic signals that simulate a disruption building  │
│  - Does the system detect, assess, and recommend correctly?      │
│  - Like fire drills for the prediction system                    │
│  - Red team: try to fool the system with misleading signals      │
│                                                                   │
│  LEVEL 4: REAL-WORLD OUTCOMES (ongoing tracking)                 │
│  - Track every alert issued: was the prediction correct?         │
│  - Track every disruption that occurred: did we alert in advance?│
│  - Key metrics:                                                   │
│    - Detection rate: % of real disruptions we alerted on         │
│    - Lead time: how far in advance did we alert?                 │
│    - Precision: % of alerts that corresponded to real events     │
│    - Action value: $ saved by preemptive actions taken            │
│                                                                   │
│  TARGET METRICS:                                                  │
│  - Detection rate: >80% of significant disruptions (week+ ahead) │
│  - Lead time: >3 days average warning before impact              │
│  - Precision: >80% (4 real alerts per 1 false)                   │
│  - Action value: measurable $ savings from early response        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

### Evolution & What to Build First:
```
"Given the difficulty spectrum, I'd phase this:

Phase 1 (Month 1-3): REACTIVE INTELLIGENCE
  - Rapid detection of KNOWN disruption patterns
  - Instant exposure mapping (who's affected?)
  - Manual risk assessment, automated impact simulation
  - Value: detect faster than humans scanning news

Phase 2 (Month 3-6): PREDICTIVE WEATHER & LOGISTICS
  - Weather-based disruption prediction (most tractable)
  - Port congestion forecasting (good data available)
  - Automated severity estimation
  - Value: 3-7 day advance warning for weather/logistics

Phase 3 (Month 6-12): SUPPLIER RISK SCORING
  - Financial health monitoring
  - Leading indicator models for supplier distress
  - Value: 30-90 day early warning on supplier failures

Phase 4 (12+ months): FULL PREDICTIVE PLATFORM
  - Geopolitical risk (hardest, lowest accuracy)
  - Novel event detection (pattern matching to historical analogs)
  - Autonomous preemptive actions
  - Value: comprehensive risk coverage

THE FIRST THING I'D BUILD (week 1):
A simple prototype that:
1. Monitors 5 news APIs for keywords matching our customer's top 20 suppliers
2. If match → query supply chain graph → show affected products
3. Human validates: 'Was this alert useful?'
4. Learn from 2 weeks of human feedback before building anything complex

This validates the MOST IMPORTANT ASSUMPTION: can we detect supply chain-
relevant events from public signals in a timely and useful way?
If this doesn't work, the whole system concept fails."
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Problem decomposition | Categorized disruption types, different approaches per type | ☐ |
| First principles | Risk = P(hazard) × Exposure × Impact ÷ Resilience | ☐ |
| Multi-source fusion | Correlation, confidence scoring, noise filtering | ☐ |
| Rare event handling | Analogs, scenarios, leading indicators, honest about limits | ☐ |
| Knowledge graph | Supply chain as graph, exposure query, cascade propagation | ☐ |
| Evaluation strategy | Backtesting, simulation, real-world tracking | ☐ |
| Honest about limitations | Explicitly stated what system CAN'T do | ☐ |
| Moonshot + pragmatic | Big vision but phased practical delivery | ☐ |
| Staff signals | "What I'd build first", de-risking assumptions | ☐ |
# ENHANCED: Disruption Prediction — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 6 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES (GOOGLE-SCALE)

### Signal Volume Analysis
```
MULTI-SOURCE SIGNAL INGESTION (Google-Scale Platform — 5000 customers):

┌─────────────────────────────────────────────────────────────────────────────────┐
│ Source              │ Raw Volume       │ After Filter    │ Useful Signals        │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ News (GDELT, Reuters│ 5M articles/day  │ 200K supply     │ ~2,000 events/day     │
│  AP, 50+ languages) │ (global coverage)│ chain relevant  │ (affecting entities)  │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Weather (NOAA, ECMWF│ 500M forecast    │ 50M in supply   │ ~500 severe           │
│  local met services)│ data points/day  │ chain zones     │ weather events/day    │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Financial (stock,   │ 100M data pts/day│ 5M supplier     │ ~100 signals/day      │
│  credit, filings,   │ (all exchanges,  │ specific        │ (bankruptcies, rating │
│  SEC/equivalent)    │  global)         │                 │  changes, late filings)│
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Maritime AIS (global│ 500M position    │ 10M on tracked  │ ~5,000 congestion/    │
│  vessel tracking)   │ reports/day      │ routes          │  delay signals/day    │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Satellite imagery   │ 50K images/day   │ 10K relevant    │ ~200 signals/day      │
│ (SAR, optical, IR)  │ (Sentinel, PlaS) │ (factories,ports│ (fire, flood, closure)│
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Social media (Reddit│ 50M posts/day    │ 100K relevant   │ ~500 signals/day      │
│  Twitter, forums)   │ (supply chain    │                 │ (strikes, protests,   │
│                     │  keywords)       │                 │  viral events)        │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Government/Regulatry│ 10K updates/day  │ 2K relevant     │ ~50 signals/day       │
│ (sanctions, tariffs │ (all countries)  │                 │ (policy changes)      │
│  port regulations)  │                  │                 │                       │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ Customer telemetry  │ 5B events/day    │ 50M anomalous   │ ~50K early warning    │
│ (from event system) │ (shared platform)│                 │ signals/day           │
├─────────────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ TOTAL               │ ~6B raw/day      │ ~265M filtered  │ ~58,000 signals/day   │
└─────────────────────────────────────────────────────────────────────────────────┘

SIGNAL FUNNEL (Google-Scale):
  6B raw data points → 265M potentially relevant → 58K confirmed signals
  → 10K mapped to specific customer supply chains → 2K scored high-risk
  → 500 actionable predictions/day → 100 critical alerts (≥$10M impact)
  
  REDUCTION RATIO: 12,000,000:1 (raw to critical actionable)
  
  PER CUSTOMER (5000 tenants):
  - 58K signals ÷ relevance overlap → avg 20 signals mapped per customer per day
  - Of which: 5 are new predictions, 15 are updates to existing tracked risks
  - Customer alert volume: 2-5 actionable alerts/day (CRITICAL to keep low)

CROSS-CUSTOMER INTELLIGENCE (Google's unique advantage):
  - 5000 customers × avg 10K suppliers = 50M supplier relationships
  - Many suppliers shared: avg supplier appears in 15 customer networks
  - If Supplier X shows distress signal → alert ALL 15 customers instantly
  - NO SINGLE CUSTOMER can build this — only a PLATFORM can aggregate
  - This is the moat: proprietary cross-network intelligence

COMPUTE REQUIREMENTS:
  NLP Pipeline (news/social — 55M docs/day):
  - Classification: 55M × 50ms (BERT-base) = 760 GPU-hours/day
  - Entity extraction: 200K relevant × 200ms (NER) = 11 GPU-hours/day
  - Summarization: 200K × 500ms (Gemini Flash) = 28 GPU-hours/day
  - Total NLP compute: 800 GPU-hours/day = 34 A100 GPUs running 24/7
  - Cost: ~$50K/month (GPUs) + $20K/month (Gemini API for summarization)

  Satellite/Image Analysis:
  - 50K images × 5 seconds (VLM analysis) = 70 GPU-hours/day
  - 3 A100 GPUs dedicated + burst capacity
  - Cost: ~$10K/month

  Maritime/Geospatial:
  - 500M AIS points → streaming aggregation (port congestion, route delays)
  - 20 Dataflow workers (geo-indexed streaming joins)
  - Cost: ~$15K/month

  Knowledge Graph Operations:
  - 10B-node graph (all supply chain entities globally)
  - Impact propagation queries: BFS/DFS with 3-hop limit
  - 10K propagation queries/day × avg 500ms = 1.4 hours compute
  - BUT: needs fast graph traversal → Neo4j cluster or Spanner graph
  - Graph update: 50K entity changes/day (new relationships, removed entities)
  - Cost: ~$80K/month (Spanner graph at this scale)

  Risk Scoring & Prediction:
  - 58K signals × risk scoring model (ensemble) = 500ms each = 8 GPU-hours/day
  - Impact simulation (Monte Carlo): 2K high-risk × 1000 simulations × 100ms
    = 55 GPU-hours for daily simulation run
  - LLM reasoning for top 100 critical predictions: $0.10 each = $10/day (trivial)
  - Cost: ~$20K/month

TOTAL COMPUTE COST:
┌──────────────────────────────────────────────────────────────────┐
│ Component              │ Monthly Cost  │ % of Total              │
├────────────────────────┼───────────────┼─────────────────────────┤
│ NLP/Signal Processing  │ $70K          │ 18%                     │
│ Satellite Analysis     │ $10K          │ 3%                      │
│ Maritime/Geospatial    │ $15K          │ 4%                      │
│ Knowledge Graph        │ $80K          │ 21%                     │
│ Risk Scoring/Predict   │ $20K          │ 5%                      │
│ Data Acquisition       │ $120K         │ 31% (AIS, sat, news)    │
│ Storage (graph + events│ $50K          │ 13%                     │
│ Infrastructure/GKE     │ $20K          │ 5%                      │
├────────────────────────┼───────────────┼─────────────────────────┤
│ TOTAL                  │ ~$385K/month  │                         │
│ Per customer           │ ~$77/month    │ (extremely cheap!)      │
│ Revenue/customer       │ $50-500K/year │                         │
└──────────────────────────────────────────────────────────────────┘

KEY INSIGHT: Disruption prediction is CHEAP to run (sub-$400K/month)
but ENORMOUSLY VALUABLE ($50B+ in prevented losses across customers).
The hard part is not compute — it's data acquisition and model accuracy.
This is the ultimate high-margin Google product: low COGS, massive value.

UNIT ECONOMICS:
- Platform cost: $4.6M/year
- Revenue (5000 × avg $100K): $500M/year
- Gross margin: 99%+ (!!)
- The value proposition: prevent ONE Suez-scale event for ONE customer
  and the product pays for itself for 10 years
```

---


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
