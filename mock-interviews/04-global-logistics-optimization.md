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
# ENHANCED: Global Logistics Optimization — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 4 — Read alongside the main file

---

## RIGOROUS SCALE ESTIMATES

### Route Optimization Compute
```
PROBLEM SIZING:
- 10K shipments/day needing routing
- Route network: 1,000 nodes, 10K edges
- Time-expanded graph (168 hours × hourly granularity): 
  1,000 × 168 = 168K time-nodes, ~500K time-edges
- Each shipment needs: 10-50 candidate routes evaluated

SINGLE SHIPMENT ROUTING:
  Graph: 168K nodes, 500K edges
  Algorithm: Constrained A* (or Dijkstra with constraints)
  Time: O((V + E) × log V) = O(500K × 18) ≈ 9M operations
  At 1 GHz effective throughput: ~9ms per routing request
  With constraint checking overhead: ~50-100ms per shipment
  
BATCH ROUTING (nightly, all planned shipments):
  8,000 planned shipments × 100ms = 800 seconds = 13 minutes (sequential)
  With 50 parallel workers: 16 seconds
  This is FAST. Single-shipment routing is not the bottleneck.

FLEET OPTIMIZATION (THE HARD PROBLEM):
  Vehicle Routing Problem (VRP) with:
  - 200 vehicles, 8K shipments, time windows, capacity, multi-modal
  - NP-hard → can't solve optimally at this scale
  - Heuristic: Large Neighborhood Search (LNS)
    - Start with greedy assignment
    - Iteratively destroy + repair segments
    - 1000 iterations × 8K evaluations = 8M evaluations
    - With efficient data structures: ~30 minutes to converge
    - Quality: within 5-10% of optimal (vs 30%+ for greedy alone)
    
CONSOLIDATION OPTIMIZATION:
  - 8K shipments → group compatible ones into containers
  - Bin packing problem: weight + volume constraints
  - Greedy + local search: O(n²) = 64M comparisons
  - With sorting pre-optimization: O(n log n) = manageable
  - Time: 2-5 minutes for all 8K shipments
```

### Infrastructure Requirements
```
┌────────────────────────────────────────────────────────────────────┐
│ Component              │ Specification           │ Cost/month        │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ Graph Database         │ Neo4j (1K nodes, 10K    │ $2K               │
│ (route network)        │ edges, frequently read) │ (or JanusGraph    │
│                        │                         │ on Bigtable: $1K) │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ Time-Expanded Graph    │ In-memory (Redis or     │ $500              │
│ (rebuilt hourly)       │ application memory)     │                   │
│                        │ 500K edges × 100B = 50MB│                   │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ Optimization Engine    │ 4 high-CPU instances    │ $3K               │
│ (OR-Tools / custom)    │ 32 cores, 64GB each     │                   │
│                        │ For VRP + consolidation │                   │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ Rate Engine            │ Redis (carrier rates,   │ $800              │
│ (pricing lookups)      │ 100K rate entries, TTL) │                   │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ ML Models              │ 2 GPU instances         │ $2K               │
│ (transit prediction,   │ (transit time, disruption│                  │
│  disruption risk)      │  risk inference)        │                   │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ Shipment Tracking DB   │ Bigtable or DynamoDB    │ $1.5K             │
│ (10K active shipments) │ 500K events/day writes  │                   │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ Event Bus (disruptions)│ Kafka (3 brokers)       │ $1.5K             │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ API + Orchestration    │ 4 app server instances  │ $1K               │
├────────────────────────┼─────────────────────────┼───────────────────┤
│ TOTAL                  │                         │ ~$12K/month       │
└────────────────────────────────────────────────────────────────────┘
```

---

## ALTERNATIVE ARCHITECTURES

### Approach A: OR-Tools + Heuristic Solver (Traditional Operations Research)
```
DESIGN:
  Route network as weighted graph
  OR-Tools constraint programming solver
  Custom heuristics for domain-specific constraints

SOLVER STACK:
  1. Pre-solver: eliminate infeasible edges (customs, mode incompatibility)
  2. Initial solution: greedy nearest-neighbor or savings algorithm
  3. Improvement: LNS (destroy 20% of solution, re-optimize that portion)
  4. Termination: time limit (5 min for batch, 5 sec for real-time)

PROS:
+ Deterministic (same input → same output, important for audit)
+ Provably feasible (constraint satisfaction guaranteed)
+ Fast for single-shipment routing (<100ms)
+ Handles hard constraints precisely (weight, temperature, customs)
+ Well-understood quality bounds (% from optimal)
+ No training data needed (works day one)

CONS:
- Doesn't learn from historical outcomes (static optimization)
- Edge weights are point estimates (no uncertainty modeling)
- Can't handle soft constraints well (strong preference vs hard rule)
- Custom development needed for every new constraint type
- VRP at scale (>10K shipments) needs engineering for performance

WHEN TO CHOOSE: 
  Primary approach. This is the backbone of any logistics optimization system.
  ML augments this; ML doesn't replace this.
```

### Approach B: ML-Enhanced Optimization (Hybrid) — MY CHOICE
```
DESIGN:
  OR-Tools solver (Approach A) PLUS ML models that improve edge weights

  ML provides BETTER INPUTS to the optimizer:
  1. Transit time prediction: "This route usually takes 14 days, 
     but given current port congestion, predict 17 days (±2)"
  2. Disruption probability: "30% chance of delay on Shanghai-LA route 
     due to weather pattern"
  3. Cost prediction: "Spot rate likely to drop 10% next week — delay booking"
  4. Carrier reliability: "Carrier X has 92% on-time for this lane, 
     Carrier Y has 87%"

  These ML predictions become edge weights in the graph:
  edge_cost = transport_cost + α × time_risk + β × disruption_prob × impact

SOLVER + ML INTERACTION:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  STEP 1: ML models predict edge attributes                       │
│    transit_time_model(route_features) → predicted_time           │
│    disruption_model(route, weather, news) → risk_score           │
│    cost_model(lane, time, demand) → predicted_cost               │
│                                                                   │
│  STEP 2: Edge weights computed from ML predictions                │
│    weight = λ₁×predicted_cost + λ₂×predicted_time                │
│             + λ₃×risk_score × potential_impact                   │
│                                                                   │
│  STEP 3: OR-Tools optimizes on these weights                     │
│    Exact optimization with ML-enhanced weights                   │
│                                                                   │
│  RESULT: Optimal routes that account for PREDICTED conditions    │
│          rather than just historical averages                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

PROS:
+ Best of both worlds: mathematical optimality + learned predictions
+ Adapts to changing conditions (ML models retrain on latest data)
+ Risk-aware routing (avoid risky routes even if nominally cheaper)
+ Still deterministic given the same predictions
+ Graceful degradation: if ML fails, fall back to historical averages

CONS:
- More complex (two systems to maintain: solver + ML pipeline)
- ML model errors propagate into routing decisions
- Need feedback loop: did predicted transit time match actual?
- Model monitoring required (drift detection on predictions)
- 3-6 months to build vs 1-2 months for pure OR approach

WHEN TO CHOOSE: When you have historical data and the value of better 
predictions justifies the ML investment. For supply chain at X's scale: definitely.
```

### Approach C: Reinforcement Learning (Research-Grade)
```
DESIGN:
  RL agent learns routing policy from experience
  State: current network conditions, pending shipments
  Action: route assignment for each shipment
  Reward: negative of (cost + lateness penalties)

ARCHITECTURE:
  - Environment: discrete event simulation of supply chain
  - Agent: PPO or SAC policy network
  - Training: millions of simulated episodes
  - Deployment: policy network inference for routing decisions

PROS:
+ Can discover non-obvious strategies (e.g., "slightly longer route 
  avoids congestion that would delay 10 other shipments")
+ Handles dynamic, sequential decision-making naturally
+ Can optimize for long-term objectives (not just immediate cost)
+ Potentially superhuman performance on complex scenarios

CONS:
- Requires high-fidelity simulator (VERY expensive to build)
- Training instability (RL is notoriously finicky)
- Non-interpretable (can't explain WHY a route was chosen)
- Catastrophic actions during exploration (need safe constraints)
- Enterprise customers won't trust black-box routing decisions
- 12-18 months of research before production-viable
- Simulator must faithfully represent real-world constraints

WHEN TO CHOOSE: Never as the primary approach for enterprise logistics
(trust/interpretability requirements). HOWEVER, useful as a research tool:
- Train RL in simulation → discover heuristics → encode as rules
- Use RL for "what-if" scenario planning (not live routing)
```

### My Architecture Decision:
```
USE APPROACH B (ML-Enhanced OR):
- OR-Tools solver provides the mathematical backbone (trust, optimality)
- ML models provide better inputs (adapt to conditions)
- This is the RIGHT level of ML for this problem:
  - ML predicts ATTRIBUTES (transit time, risk) — interpretable
  - Solver optimizes DECISIONS (which route) — provably feasible
  - User sees: "Route via Oakland recommended because model predicts 
    3-day delay at LA port (72% confidence based on current vessel queue)"
  - This is EXPLAINABLE and TRUSTWORTHY

EVOLUTION:
- Year 1: Pure OR-Tools with historical averages as edge weights
- Year 2: Add transit time prediction model (biggest value driver)
- Year 3: Add disruption risk model, dynamic cost prediction
- Year 4+: Explore RL for scenario planning and long-term strategy
```

---

## TRADE-OFF MATRICES

### Optimization Objective Trade-offs
```
MULTI-OBJECTIVE OPTIMIZATION: Can't minimize everything simultaneously.

Objective dimensions:
1. Cost (transport + handling + duties)
2. Time (total transit hours)
3. Risk (probability of delay or damage)
4. Carbon (CO2 emissions)

PARETO FRONTIER EXAMPLE:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Cost ↑                                                          │
│   │           ×C (air: $12K, 2 days)                            │
│   │                                                              │
│   │                                                              │
│   │     ×B (truck+rail: $5K, 8 days)                            │
│   │                                                              │
│   │                                                              │
│   │  ×A (ocean+rail: $3K, 20 days)                              │
│   │                                                              │
│   └──────────────────────────────────── Time →                   │
│                                                                   │
│  PARETO OPTIMAL: All three are Pareto-optimal (can't improve one │
│  dimension without worsening another).                            │
│                                                                   │
│  HOW TO CHOOSE: User specifies constraint, system optimizes:     │
│  "Minimize cost subject to: arrive by June 25" → picks A or B    │
│  "Minimize time subject to: budget < $6K" → picks B              │
│  "Minimize risk subject to: cost < $5K" → evaluates reliability  │
│                                                                   │
│  STAFF INSIGHT: Present 3-5 Pareto-optimal options to user.      │
│  Don't just show "the best" — show the TRADE-OFFS.               │
│  Let the human make the value judgment.                          │
└─────────────────────────────────────────────────────────────────┘
```

### Consolidation Trade-offs
```
PROBLEM: Should we consolidate shipments?
(Put multiple shipments in the same container/vehicle)

BENEFITS:
- Cost per shipment drops 30-50% (shared transport cost)
- Better utilization (40% of containers ship partially empty)
- Fewer total shipments → less coordination overhead

COSTS:
- Delay: Wait for enough shipments to fill container (hours to days)
- Risk: One delayed shipment delays all co-loaded shipments
- Flexibility: Harder to reroute individual shipments
- Complexity: Loading order matters, compatibility constraints

DECISION FRAMEWORK:
┌─────────────────────────────────────────────────────────────────┐
│ CONSOLIDATE when:                                                │
│ - Time slack > 2 days (can afford to wait for grouping)         │
│ - Destination cluster within 50km (same general direction)      │
│ - Shipments are compatible (no hazmat with food, etc)           │
│ - Cost savings > $500 per consolidated group                    │
│                                                                  │
│ DON'T CONSOLIDATE when:                                         │
│ - Urgent/time-critical (every hour matters)                     │
│ - High-value + perishable (risk of co-loaded delays)            │
│ - Different customs regimes (creates paperwork complexity)       │
│ - Customer requires dedicated shipment (contractual)            │
└─────────────────────────────────────────────────────────────────┘
```

### Re-Routing Decision Trade-offs
```
WHEN TO RE-ROUTE vs WAIT:

THE MATH:
  reroute_cost = new_route_cost - sunk_cost_of_current_route
                 + rebooking_fees + cancellation_penalties
  
  wait_cost = P(disruption_continues) × (SLA_penalty + inventory_cost × delay_days)
              + opportunity_cost_of_uncertainty

  DECISION: reroute if reroute_cost < expected_wait_cost

EXAMPLE:
  Shipment: $200K electronics, Shanghai → Chicago
  Current: Stuck at congested port (expected 5-day delay, 60% probability)
  Alternative: Reroute via air ($4K extra, arrives on time)
  
  wait_cost = 0.6 × ($10K SLA penalty + $200K × 8%/year × 5/365) = 
              0.6 × ($10K + $219) = $6,131
  
  reroute_cost = $4,000 (air freight premium) + $500 (rebooking) = $4,500
  
  $4,500 < $6,131 → REROUTE (save ~$1,600 in expected value)

COMPLICATION: This is a STOCHASTIC decision. Key uncertainties:
1. How long will disruption last? (predict with ML model)
2. Will the alternative route also be affected? (correlation risk)
3. Will more capacity become available if we wait? (market dynamics)

STAFF INSIGHT: Frame re-routing as expected value calculation, not 
deterministic comparison. Present uncertainty ranges to the planner.
"Reroute saves $1,600 in expectation (range: lose $2K to save $6K)."
```

---

## GRAPH DATA STRUCTURE DESIGN

### Why Time-Expanded Graph (TEG)
```
PROBLEM WITH STATIC GRAPH:
  Static: Shanghai → LA, cost=$3K, time=14 days
  Reality: 
    - Ship departs Mon/Wed/Fri
    - If I arrive at port on Tuesday, I wait until Wednesday
    - That 1-day wait changes everything downstream

TIME-EXPANDED GRAPH (TEG):
  Create a copy of each node for each time step:
  
  Shanghai_Mon-8am → (wait) → Shanghai_Mon-4pm (departure)
  Shanghai_Mon-4pm → (ocean, 14 days) → LA_Mon+14_8am (arrival)
  Shanghai_Tue-8am → (wait) → Shanghai_Wed-4pm (next departure)
  
  Now "find shortest path from Shanghai_today to Chicago_before_deadline"
  is a standard graph problem (Dijkstra works!)

MEMORY:
  1,000 locations × 168 hours = 168,000 time-nodes
  Edges: ~500K (transport + waiting + transfer)
  Node size: 64 bytes (location, time, capacity)
  Edge size: 128 bytes (mode, carrier, cost, capacity, constraints)
  Total: 168K×64 + 500K×128 = 75 MB
  → Fits easily in memory. Rebuilt every hour with fresh data.

TRADE-OFF: GRANULARITY
  Hourly granularity: 168K nodes/week, captures all schedules
  15-min granularity: 672K nodes/week, better for last-mile timing
  Daily granularity: 7K nodes/week, too coarse (misses schedules)
  
  MY CHOICE: Hourly for long-haul legs, 15-min for last-mile/local
```

### Alternative: Dynamic Graph Without Time Expansion
```
APPROACH: Keep static graph but attach schedules to edges as attributes

  Edge: Shanghai → LA
    Schedules: [Mon 4pm, Wed 4pm, Fri 4pm]
    Transit: 14 days
    Capacity: 2000 TEU per sailing

  Routing algorithm: Modified Dijkstra that checks:
    "Can I depart at this time? If not, when's the next departure?"
    
  PROS: Less memory (10K edges vs 500K), simpler to update
  CONS: More complex routing algorithm, harder to parallelize,
        non-standard Dijkstra (correctness harder to verify)

WHY I PREFER TEG: Standard algorithms work unchanged (Dijkstra, A*).
Correctness is easier to verify. Memory is not a constraint at this scale.
The computational clarity is worth the extra memory.
```

---

## REAL-TIME DECISION LATENCY ANALYSIS

```
SCENARIO: Port closure detected. Re-route 32 affected shipments.

┌──────────────────────────────────────────────┬──────────────────┐
│ Step                                         │ Time             │
├──────────────────────────────────────────────┼──────────────────┤
│ 1. Disruption event received (Kafka)          │ <100ms           │
│ 2. Impact query: "which shipments affected?"  │ 200ms (DB query) │
│ 3. Triage: classify urgency per shipment      │ 50ms (rules)     │
│ 4. Graph update: remove disrupted edges        │ 10ms (in-memory) │
│ 5. Parallel re-route (32 shipments × 100ms)   │ 200ms (parallel) │
│ 6. Cost-benefit analysis per shipment          │ 100ms (math)     │
│ 7. Format recommendations                     │ 50ms             │
│ 8. Push notification to planners              │ 100ms            │
├──────────────────────────────────────────────┼──────────────────┤
│ TOTAL                                         │ ~800ms           │
└──────────────────────────────────────────────┴──────────────────┘

THIS IS SUB-SECOND! The real latency is human decision time, not compute.

DESIGN INSIGHT: Since compute is fast, we can afford to:
1. Pre-compute alternatives for AT-RISK shipments (before disruption hits)
2. Run Monte Carlo simulations (100 scenarios × 100ms = 10 seconds)
3. Present risk-adjusted recommendations with confidence intervals

PRE-COMPUTATION STRATEGY:
- Every hour: identify top 100 at-risk shipments (ML risk model)
- Pre-compute alternative routes for each
- If disruption hits: instantly show pre-computed alternatives
- Dramatically reduces perceived response time to ZERO (already computed)
```

---

## MULTI-REGION DEPLOYMENT CONSIDERATIONS

```
WHY MULTI-REGION MATTERS FOR LOGISTICS:
- Logistics is inherently global (shipments span continents)
- Regulatory: some data can't leave certain regions (EU → EU only)
- Latency: Asia planners need fast access to Asia route data
- Resilience: one region failing can't stop global routing

DEPLOYMENT ARCHITECTURE:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  REGION: Americas (GCP us-central1)                              │
│    - Route graph: Americas nodes + cross-region edges            │
│    - Active shipments: Americas origin/destination               │
│    - Solver instances: for Americas routing                      │
│                                                                   │
│  REGION: EMEA (GCP europe-west1)                                 │
│    - Route graph: EMEA nodes + cross-region edges                │
│    - Active shipments: EMEA origin/destination                   │
│    - Solver instances: for EMEA routing                          │
│    - GDPR-compliant storage                                      │
│                                                                   │
│  REGION: APAC (GCP asia-east1)                                   │
│    - Route graph: APAC nodes + cross-region edges                │
│    - Active shipments: APAC origin/destination                   │
│    - Solver instances: for APAC routing                          │
│                                                                   │
│  GLOBAL LAYER (replicated everywhere):                           │
│    - Cross-region route graph (ocean lanes, air routes)          │
│    - Global disruption feed                                      │
│    - Rate data (replicated from each carrier's home region)      │
│                                                                   │
│  CROSS-REGION ROUTING:                                           │
│    Shanghai (APAC) → Chicago (Americas):                          │
│    1. APAC solver handles Shanghai → departure port              │
│    2. Global layer handles ocean crossing                        │
│    3. Americas solver handles arrival port → Chicago             │
│    4. Orchestrator stitches the segments together                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

TRADE-OFF: 
  Centralized (one region does all routing):
    Pro: Simple, no cross-region coordination
    Con: High latency for distant regions, single point of failure
    
  Fully distributed (each region independent):
    Pro: Low latency, high resilience
    Con: Cross-region routes require coordination, data sync complexity
    
  MY CHOICE: Hybrid (region-local for domestic, coordinated for international)
```
