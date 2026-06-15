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

### Interviewer's Likely Answer:
- All categories in scope (but prioritize by feasibility)
- Platform serving many companies (multi-tenant)
- 1-14 day prediction horizon is most actionable
- Significant disruptions only (>$100K potential impact)
- External data acquisition: yes, reasonable budget
- False positive tolerance: <20% (5 false alerts per 1 real disruption)
- Supply chain graph available per customer

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
