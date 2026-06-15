# System Design Interview Framework

## The 4-Phase Approach (45-50 minutes)

### Phase 1: Requirements & Scoping (5-8 minutes)

**This is where Staff engineers differentiate themselves.**

#### Functional Requirements
Ask: "What does this system need to DO?"
- Core use cases (2-3 max for a 45-min interview)
- User personas and workflows
- Input/output contracts

#### Non-Functional Requirements
Ask: "What qualities must it have?"
- **Scale**: Users, requests/sec, data volume, growth rate
- **Latency**: p50, p95, p99 targets
- **Availability**: 99.9%? 99.99%? What's the cost of downtime?
- **Consistency**: Strong vs eventual? Where does it matter?
- **Durability**: Can we lose data? Recovery time objectives?

#### Constraints & Assumptions
- Budget/team size (startup vs big company)
- Existing infrastructure (cloud provider, tech stack)
- Regulatory requirements (GDPR, SOX for supply chain)
- Timeline (MVP vs production-grade)

**Staff-Level Move**: Don't just ask questions — propose constraints and validate them.
- "I'm going to assume we need 99.9% availability since supply chain disruptions cost $X/hour. Does that align?"
- "Given this is enterprise software, I'll assume we need SOC 2 compliance and audit logging."

#### Back-of-Envelope Calculations
Always estimate early:
```
Users: 10,000 enterprise users
Peak QPS: 10,000 users × 10 requests/hour = ~28 QPS (bursty to 5x = 140 QPS)
Storage: 1M supply chain events/day × 1KB avg = 1GB/day = 365GB/year
LLM calls: 500 queries/hour × $0.01/query = $120/day
```

---

### Phase 2: High-Level Design (10-12 minutes)

#### Start with the API
Define the contract first:
```
POST /api/v1/forecast
  Request: { sku_ids: [], horizon_days: 30, confidence_level: 0.95 }
  Response: { forecasts: [{ sku_id, date, predicted_demand, confidence_interval }] }
```

#### Draw the Architecture
Components to consider:
- **Client Layer**: Web app, mobile, API consumers
- **API Gateway / Load Balancer**: Rate limiting, auth, routing
- **Application Services**: Business logic, orchestration
- **Data Layer**: Databases, caches, message queues
- **AI/ML Layer**: Model serving, feature stores, embedding services
- **Infrastructure**: Monitoring, logging, deployment

#### Data Flow
Trace the happy path end-to-end:
1. User submits request →
2. API gateway authenticates and rate-limits →
3. Service orchestrates data retrieval →
4. ML model generates prediction →
5. Result is cached and returned

**Staff-Level Move**: Identify the 2-3 hardest problems in your design and call them out.
- "The interesting challenges here are: (1) keeping predictions fresh as new data arrives, (2) handling the cold-start problem for new SKUs, and (3) explaining predictions to non-technical users."

---

### Phase 3: Deep Dive (20-25 minutes)

**This is where interviews are won or lost at Staff level.**

Pick 2-3 components to go deep on. Prioritize:
1. The component most critical to the system's success
2. The component with the most interesting trade-offs
3. The component the interviewer shows interest in

#### For each deep dive:
- **Data model**: Schema design, indexing strategy, partitioning
- **Algorithm/approach**: Why this algorithm? What are alternatives?
- **Failure modes**: What happens when this component fails?
- **Scale**: How does this component scale? What's the bottleneck?
- **Trade-offs**: What did you give up? Why is that acceptable?

#### Trade-off Framework
For every decision, articulate:
```
Decision: Use eventual consistency for inventory counts
Why: Strong consistency would require distributed transactions,
     adding 200ms latency and reducing throughput 10x
Trade-off: Users may see stale counts for up to 5 seconds
Mitigation: Show "last updated" timestamp; use strong consistency
           for order placement (where correctness matters)
```

---

### Phase 4: Wrap-Up & Extensions (5-8 minutes)

#### Operational Concerns
- **Monitoring**: What metrics matter? (latency percentiles, error rates, ML model drift)
- **Alerting**: What pages someone at 2am?
- **Deployment**: Canary → staged → full rollout
- **Testing**: How do you test a distributed system?

#### Evolution & Scale
- "If we 10x traffic, what breaks first?"
- "How would we add multi-region support?"
- "What would a v2 look like?"

#### Staff-Level Closing Moves
- Summarize the key trade-offs you made and why
- Identify what you'd investigate further with more time
- Mention organizational/team implications ("This needs a dedicated ML platform team")

---

## Common Pitfalls to Avoid

| Pitfall | Fix |
|---------|-----|
| Jumping into solution immediately | Always start with requirements |
| Designing for infinite scale from day 1 | Start simple, identify scale triggers |
| Only one option considered | Present 2-3 approaches, pick with reasoning |
| Handwaving trade-offs | Quantify: "This adds 50ms but saves $X/month" |
| Ignoring failure modes | Every component can fail; design for it |
| Not involving the interviewer | "I'm thinking about X. Would you like me to go deeper or move on?" |
| Designing a single box | At Staff level, think multi-service, multi-team |

---

## The "Staff Signal" Checklist

Use this as a mental checklist during your interview:

- [ ] I defined the problem before solving it
- [ ] I made explicit assumptions and validated them
- [ ] I did back-of-envelope math
- [ ] I considered at least 2 approaches before picking one
- [ ] I quantified trade-offs (not just "faster" but "200ms faster at cost of...")
- [ ] I identified the hardest problems and went deep on them
- [ ] I considered operational concerns (monitoring, deployment, failure)
- [ ] I discussed how the system evolves over time
- [ ] I communicated clearly and structured my thinking
- [ ] I drove the conversation while remaining collaborative
