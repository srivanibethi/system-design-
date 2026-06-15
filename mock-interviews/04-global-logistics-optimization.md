# Mock Interview 4: Global Logistics Route Optimization

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "Design a system that optimizes shipping routes and logistics decisions for a global supply chain network. Think multi-modal (truck, rail, ocean, air), across dozens of countries, with constraints like customs, perishability, cost, and time. The system should recommend optimal routes and adapt when disruptions occur."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Scope:**
1. "Are we optimizing for a single company's logistics, or building a platform for multiple companies?"
2. "What modalities: just truck? Or multi-modal (truck → port → ocean → port → truck)?"
3. "Scale: how many shipments per day? How many possible routes?"
4. "What's the optimization objective: minimize cost? Minimize time? Balance of both?"

**About Constraints:**
5. "Hard constraints I should know about: customs regulations, hazmat, temperature control, weight limits, driver hours-of-service?"
6. "Do we need to handle dynamic re-routing? E.g., port closure mid-shipment?"
7. "SLA commitments: guaranteed delivery dates that can't be violated?"
8. "Consolidation: can we combine multiple shipments on the same vehicle?"

**About Data:**
9. "What data do we have about routes: historical transit times, carrier performance, real-time traffic/weather?"
10. "How often do route costs change? (Fuel prices, carrier rates, spot market)"
11. "Do we have real-time visibility of vehicle positions?"

**About Users:**
12. "Who uses this: logistics planners? Or is it fully automated?"
13. "Do they need to see alternatives (route A vs B vs C) or just the best recommendation?"
14. "How far in advance are shipments planned vs on-demand?"

### Expected Answers:
- Multi-modal global logistics for a single large enterprise (expandable later)
- 10,000 shipments/day, routes can span 5+ countries
- Optimize for cost with delivery time constraints (SLA)
- Must handle disruptions with real-time re-routing
- Logistics planners review and approve recommendations
- Mix of planned (80%) and urgent (20%) shipments

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me frame the problem:

SCALE:
- 10K shipments/day needing route optimization
- Each route has 10-50 possible options (mode × carrier × path combinations)
- Route network: ~1000 nodes (warehouses, ports, hubs, DCs), ~10K edges
- Re-routing: ~500 shipments/day affected by disruptions

COMPUTE:
- Batch optimization (nightly for next-day planned shipments): 
  8K shipments × graph search = must complete in <2 hours
- Real-time routing (urgent + re-routes):
  ~100 requests/hour × <5 seconds response time
- This is a combinatorial optimization problem that's NP-hard in general,
  but solvable with heuristics at this scale

COST FACTORS:
- Transport cost: $0.50-$5.00/km (varies by mode)
- Time cost: $X/hour of inventory in transit (opportunity cost)
- Risk cost: probability of disruption × impact
- Penalty cost: SLA violation = $Y per day late

DATA FRESHNESS:
- Carrier rates: update daily
- Real-time: traffic, weather, port congestion (update every 15 min)
- Disruptions: immediate (event-driven)

Am I framing this correctly?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DATA & CONTEXT LAYER                               │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Route Network│  │ Real-Time    │  │ Carrier &    │  │ Disruption │ │
│  │ Graph        │  │ Conditions   │  │ Rate Data    │  │ Feed       │ │
│  │ (nodes,edges,│  │ (traffic,    │  │ (contracts,  │  │ (weather,  │ │
│  │  distances,  │  │  weather,    │  │  spot market,│  │  port close│ │
│  │  modes)      │  │  congestion) │  │  schedules)  │  │  strikes)  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
└─────────┼──────────────────┼──────────────────┼───────────────┼─────────┘
          │                  │                  │               │
          ▼                  ▼                  ▼               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      OPTIMIZATION ENGINE                                  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ ROUTE GRAPH ENGINE                                                │    │
│  │                                                                   │    │
│  │  Weighted multi-modal graph where:                                │    │
│  │  - Nodes = locations (warehouses, ports, hubs, customers)         │    │
│  │  - Edges = transport legs (mode, carrier, capacity, schedule)     │    │
│  │  - Edge weights = f(cost, time, risk, CO2) — DYNAMIC              │    │
│  │                                                                   │    │
│  │  Example edge: Chicago_WH → LA_Port                               │    │
│  │    Mode: Rail | Cost: $2,400 | Time: 72h | Reliability: 95%      │    │
│  │    Mode: Truck | Cost: $4,200 | Time: 30h | Reliability: 99%     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ SOLVER LAYER                                                      │    │
│  │                                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │    │
│  │  │ Level 1: Single Shipment Router                          │     │    │
│  │  │ - Modified Dijkstra with constraints (time windows, mode │     │    │
│  │  │   compatibility, customs requirements)                   │     │    │
│  │  │ - Returns: Top K routes with cost/time/risk trade-offs   │     │    │
│  │  │ - Latency: <500ms per shipment                           │     │    │
│  │  └─────────────────────────────────────────────────────────┘     │    │
│  │                                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │    │
│  │  │ Level 2: Fleet Optimizer (batch)                         │     │    │
│  │  │ - Vehicle Routing Problem (VRP) variant                  │     │    │
│  │  │ - Consolidation: combine compatible shipments            │     │    │
│  │  │ - Capacity: respect vehicle/container limits             │     │    │
│  │  │ - Algorithm: Constraint programming (OR-Tools) +         │     │    │
│  │  │   metaheuristics (Large Neighborhood Search)             │     │    │
│  │  │ - Latency: minutes to hours (nightly batch)              │     │    │
│  │  └─────────────────────────────────────────────────────────┘     │    │
│  │                                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │    │
│  │  │ Level 3: ML-Enhanced Predictions                         │     │    │
│  │  │ - Transit time prediction (actual vs scheduled)          │     │    │
│  │  │ - Disruption probability per route                       │     │    │
│  │  │ - Cost prediction (spot market rates)                    │     │    │
│  │  │ - Used to adjust edge weights in the graph               │     │    │
│  │  └─────────────────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ CONSTRAINT ENGINE                                                 │    │
│  │ - Hard constraints: customs (docs required), hazmat (prohibited   │    │
│  │   modes), temperature (reefer required), weight limits            │    │
│  │ - Soft constraints: preferred carriers, CO2 budget, cost caps     │    │
│  │ - SLA constraints: must arrive by date X (convert to time budget) │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   DISRUPTION RESPONSE LAYER                               │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ EVENT LISTENER (subscribes to disruption events from Kafka)       │    │
│  │                                                                   │    │
│  │ Disruption detected → Identify affected shipments →               │    │
│  │   Re-route using Level 1 solver (real-time, single shipment) →    │    │
│  │   Compare: new route vs wait-it-out →                             │    │
│  │   If savings > threshold: recommend re-route to planner           │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     USER INTERFACE LAYER                                   │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Route Recommendation UI                                              │ │
│  │ ┌───────────────────────────────────────────────────────────────┐  │ │
│  │ │ Shipment: 40ft container, Shanghai → Chicago                   │  │ │
│  │ │ SLA: Deliver by June 25                                        │  │ │
│  │ │                                                                 │  │ │
│  │ │ Option A (Recommended):                                         │  │ │
│  │ │   Shanghai → Busan (truck, 1d) → LA (ocean, 14d) →             │  │ │
│  │ │   Chicago (rail, 3d) | $4,200 | 18 days | 94% on-time          │  │ │
│  │ │                                                                 │  │ │
│  │ │ Option B (Faster):                                              │  │ │
│  │ │   Shanghai → LA (ocean express, 11d) → Chicago (truck, 2d)     │  │ │
│  │ │   | $6,800 | 13 days | 97% on-time                             │  │ │
│  │ │                                                                 │  │ │
│  │ │ Option C (Cheapest):                                            │  │ │
│  │ │   Shanghai → Long Beach (ocean slow, 18d) → Chicago (rail, 4d) │  │ │
│  │ │   | $3,100 | 22 days | 88% on-time ⚠️ SLA RISK                │  │ │
│  │ └───────────────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: The Route Graph & Solver

```
"The core algorithmic challenge: find optimal paths through a multi-modal, 
time-constrained, capacity-constrained network.

GRAPH REPRESENTATION:
┌─────────────────────────────────────────────────────────────────┐
│ Node types:                                                      │
│   - Origin warehouse (OW): where shipment starts                │
│   - Hub/Port (H): transshipment points                          │
│   - Destination (D): final delivery location                    │
│                                                                  │
│ Edge types (each has temporal + capacity dimensions):             │
│   - Road: continuous availability, travel time varies by traffic │
│   - Rail: scheduled departures (e.g., daily at 6am, 2pm)        │
│   - Ocean: scheduled sailings (e.g., weekly Wed from Shanghai)   │
│   - Air: scheduled flights (multiple daily)                      │
│                                                                  │
│ TIME-EXPANDED GRAPH:                                             │
│ Because transport has SCHEDULES, I use a time-expanded graph:    │
│                                                                  │
│   Shanghai_Mon → (ocean, departs Mon) → LA_+14days               │
│   Shanghai_Wed → (ocean, departs Wed) → LA_+14days               │
│   Shanghai_Mon → (wait 2 days) → Shanghai_Wed                    │
│                                                                  │
│ This converts the scheduling problem into a graph problem:       │
│ "Find shortest path from Shanghai_now to Chicago_beforeSLA"      │
└─────────────────────────────────────────────────────────────────┘

SOLVER ALGORITHM (Single Shipment):

Modified A* with:
  - Edge weights: cost + α×time + β×risk + γ×CO2
  - Constraints as path validation (reject invalid paths)
  - Return top-K paths (not just optimal)

def find_routes(origin, destination, shipment, sla_deadline):
    # Build time-expanded subgraph (only relevant time window)
    graph = build_time_graph(origin, destination, 
                            start=now, end=sla_deadline)
    
    # Apply constraints as edge filters
    graph.filter_edges(
        mode_allowed=shipment.allowed_modes,
        max_weight=shipment.weight,
        requires_temp_control=shipment.perishable,
        customs_docs_available=shipment.customs_cleared_for
    )
    
    # Multi-objective: find Pareto-optimal set
    routes = pareto_optimal_search(
        graph, origin, destination,
        objectives=[cost, time, risk],
        constraint=arrival_before(sla_deadline),
        k=5  # top 5 diverse routes
    )
    
    return routes

PERFORMANCE:
- Graph size: ~1000 nodes × 168 hours (weekly) = 168K time-expanded nodes
- Edges: ~10K base × schedule frequency = ~100K time-expanded edges  
- A* on this graph: <100ms per search
- 10K shipments batch: parallelize across 50 workers = 200 seconds total

WHY NOT JUST USE GOOGLE MAPS ROUTING?
- Multi-modal (Google Maps doesn't do ocean + rail + customs)
- Schedule-aware (can't just book a ship like hailing an Uber)
- Consolidation (combine shipments to fill containers)
- Enterprise constraints (preferred carriers, contractual rates)
- Cost models (complex: fuel surcharge, currency, duties, insurance)
"
```

---

### Deep Dive 2: Dynamic Re-Routing on Disruption

```
"When a disruption occurs mid-transit, we need to rapidly re-route 
affected shipments. Here's the architecture:

DISRUPTION EVENT FLOW:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Disruption: "Port of LA closed due to labor strike"              │
│       │                                                           │
│       ▼                                                           │
│  STEP 1: IMPACT ASSESSMENT (<30 seconds)                         │
│  Query: "Which in-transit shipments pass through LA port?"        │
│  → Scan active shipments table (indexed by route nodes)          │
│  → Result: 47 shipments affected                                 │
│       │                                                           │
│       ▼                                                           │
│  STEP 2: TRIAGE (<1 minute)                                      │
│  For each affected shipment, classify urgency:                    │
│  - Already past LA: no impact ✓                                  │
│  - Arriving today: CRITICAL (reroute immediately)                │
│  - Arriving this week: HIGH (reroute within hours)               │
│  - Arriving next week: MEDIUM (monitor, may resolve)             │
│  → 12 CRITICAL, 20 HIGH, 15 MEDIUM                              │
│       │                                                           │
│       ▼                                                           │
│  STEP 3: ALTERNATIVE ROUTING (<5 minutes)                        │
│  For CRITICAL + HIGH (32 shipments):                              │
│  - Remove LA port from graph (mark edge as unavailable)          │
│  - Re-run solver for each shipment                               │
│  - Alternatives: Long Beach port, Oakland port, overland route   │
│       │                                                           │
│       ▼                                                           │
│  STEP 4: COST-BENEFIT ANALYSIS                                   │
│  For each shipment, compare:                                     │
│  - Re-route cost (new carrier, mode change, expediting)          │
│  - Wait cost (SLA penalty, inventory carrying, customer impact)  │
│  - Risk of waiting (how long might disruption last?)             │
│       │                                                           │
│       ▼                                                           │
│  STEP 5: RECOMMENDATION                                         │
│  Present to logistics planner:                                    │
│  "LA port strike — 32 shipments affected.                        │
│   Recommended: Re-route 12 CRITICAL via Long Beach (+$800 avg,   │
│   saves 3 days). Monitor 20 HIGH (strike likely <48h based on    │
│   historical pattern)."                                          │
│  [Approve All] [Review Each] [Override]                          │
│       │                                                           │
│       ▼                                                           │
│  STEP 6: EXECUTION                                               │
│  On approval:                                                     │
│  - Update shipment routes in system of record                    │
│  - Notify carriers (API calls to book new legs)                  │
│  - Update ETAs for downstream stakeholders                       │
│  - Log decision for audit trail                                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

DISRUPTION DURATION PREDICTION (ML):
- Historical data: how long did similar disruptions last?
- Features: disruption type, location, season, severity indicators
- Model: survival analysis (predicts duration distribution)
- Use median predicted duration to decide: reroute vs wait

This prediction is CRITICAL for the wait-vs-reroute decision.
If strike will last 2 hours → wait. If 2 weeks → reroute everything.
"
```

---

### Deep Dive 3: Cost Modeling

```
"Accurate cost modeling is what makes route recommendations trustworthy.

COST COMPONENTS (per shipment leg):
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  TRANSPORT COST = base_rate + fuel_surcharge + accessorials       │
│    - base_rate: contracted (from rate table) or spot (API query)  │
│    - fuel_surcharge: fuel_index × rate_per_km × distance          │
│    - accessorials: pickup fee, delivery fee, detention, liftgate  │
│                                                                   │
│  TIME COST = inventory_value × carrying_rate × transit_days       │
│    - Opportunity cost of capital tied up in transit               │
│    - Higher for expensive goods (electronics > bulk materials)    │
│                                                                   │
│  RISK COST = P(disruption) × expected_impact                     │
│    - P(disruption): from ML model per route + conditions          │
│    - expected_impact: SLA penalty + customer churn + expedite cost│
│                                                                   │
│  HANDLING COST = terminal_fees + customs_brokerage + insurance    │
│    - Each transshipment point adds handling cost                  │
│    - More stops = more handling cost + more risk                  │
│                                                                   │
│  TOTAL COST = Σ(transport + time + risk + handling) over all legs │
│                                                                   │
│  CONSTRAINTS (hard):                                              │
│  - Must arrive by SLA deadline                                    │
│  - Must use reefer if perishable                                  │
│  - Must clear customs (documentation ready)                      │
│  - Cannot exceed weight/volume limits                             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

RATE DATA MANAGEMENT:
- Contract rates: loaded from ERP, updated quarterly
- Spot rates: API calls to carrier marketplaces (DAT, Freightos)
- Rate cache: 1-hour TTL for spot rates (prices change frequently)
- ML prediction: forecast next-week spot rates for planning ahead

Example total cost calculation:
  Shanghai → Chicago (40ft container, electronics, $200K value):
  
  Leg 1: Shanghai → LA (ocean, 14 days)
    Transport: $2,800 (contract rate)
    Time: $200K × 8%/year × 14/365 = $614
    Risk: 5% × $5K penalty = $250
    Handling: $400 (port fees)
    Subtotal: $4,064
    
  Leg 2: LA → Chicago (rail, 3 days)
    Transport: $1,200
    Time: $200K × 8%/year × 3/365 = $132
    Risk: 2% × $5K = $100
    Handling: $200 (terminal)
    Subtotal: $1,632
    
  TOTAL: $5,696 | 17 days | 93% on-time probability
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

```
"Key trade-offs in this system:

1. OPTIMALITY vs SPEED:
   - True optimal (integer programming) might take hours for fleet-level
   - My choice: exact for individual routes (<500ms), heuristic for fleet
   - Good enough in real-time > perfect answer too late

2. RE-ROUTE FREQUENCY vs STABILITY:
   - Re-route too often → operational chaos, carrier penalties
   - Re-route too rarely → miss cost savings
   - My choice: re-evaluate on disruption events OR when potential savings 
     exceed threshold (>15% cost reduction)

3. CENTRALIZED vs DISTRIBUTED OPTIMIZATION:
   - Centralized: globally optimal, complex, single point of failure
   - Distributed: each region optimizes locally, may miss global optima
   - My choice: centralized solver with regional caches for real-time

Evolution path:
- V1: Single-shipment routing with alternatives (ship fast, learn)
- V2: Consolidation and fleet optimization (cost savings)
- V3: Predictive disruption avoidance (proactive re-routing)
- V4: Autonomous logistics (system executes without human approval)"
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Problem modeling | Time-expanded graph, multi-modal, scheduling | ☐ |
| Algorithm choice | A*/Dijkstra for single, VRP/OR-Tools for fleet | ☐ |
| Constraint handling | Hard vs soft constraints, customs, temp, weight | ☐ |
| Disruption response | Impact assessment → triage → re-route → execute | ☐ |
| Cost model | Multi-factor (transport + time + risk + handling) | ☐ |
| Real-time vs batch | Dual-speed: batch for planning, real-time for urgency | ☐ |
| User experience | Present options with trade-offs, not just "best" answer | ☐ |
| Staff signals | Phased evolution, stability vs optimality trade-off | ☐ |
