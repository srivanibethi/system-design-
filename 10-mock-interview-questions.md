# Mock Interview Questions (Self-Practice)

## Instructions
- Set a timer for 45 minutes per question
- Talk out loud (or write on paper/whiteboard)
- Use the framework from `01-framework-and-methodology.md`
- After each attempt, review the key points below

---

## TIER 1: Most Likely for This Role (Practice These First)

### Q1: Design an Enterprise Knowledge Base with AI Q&A
**Prompt**: "Our customers have thousands of documents — SOPs, contracts, supplier reviews, logistics manuals. Design a system that lets them search and ask questions in natural language."

**Key points to hit**:
- Multi-tenant data isolation (each enterprise customer's docs are private)
- Ingestion pipeline (PDF/Word/email → chunking → embedding → vector store)
- Hybrid retrieval (vector + keyword + metadata filtering)
- Answer quality (citations, confidence scores, hallucination prevention)
- Scale: 100 enterprises × 100K docs each × concurrent users
- Feedback loop: thumbs up/down → improve retrieval quality over time
- Access control: not everyone in a company should see all documents
- Freshness: when a document is updated, how quickly does the search reflect it?

**Staff-level differentiators**:
- Propose evaluation framework (offline + online metrics)
- Discuss cold start (new customer with zero feedback data)
- Consider the human workflow: what does the user do AFTER getting an answer?
- Multi-modal: some documents have tables, images, charts

---

### Q2: Design a Demand Forecasting System
**Prompt**: "Design a system that predicts demand for 100K SKUs across 500 locations, with daily granularity, looking 30 days ahead."

**Key points to hit**:
- Feature engineering (lag features, calendar, promotions, weather, economic indicators)
- Model architecture (ensemble: statistical + ML + foundation model)
- Hierarchical forecasting (total → category → SKU; reconciliation)
- Batch training pipeline (nightly retrain on latest data)
- Serving: pre-computed batch predictions + on-demand refresh capability
- Monitoring: forecast accuracy tracking, drift detection, alert on degradation
- Explainability: why is demand predicted to spike next Tuesday?
- Cold start: new product with no history

**Staff-level differentiators**:
- Discuss the organizational impact: who consumes forecasts? How do they use them?
- Feedback loop: did actual demand match prediction? Automated retraining triggers.
- Business metrics, not just ML metrics (inventory cost saved, stockout reduction)
- Probabilistic forecasts (confidence intervals) vs point estimates

---

### Q3: Design a Real-Time Supply Chain Event Processing System
**Prompt**: "We receive millions of events daily from warehouses, carriers, and stores. Design a system that processes these events in real-time to detect anomalies and trigger actions."

**Key points to hit**:
- Ingestion: heterogeneous sources (APIs, webhooks, file drops, IoT)
- Event schema: standardization, validation, enrichment
- Stream processing: Kafka + Flink for windowed aggregations
- Anomaly detection: statistical (Z-score) + ML (isolation forest)
- Action engine: rule-based + ML-based decision making
- Alerting: routing to right person, dedup, escalation
- Scale: 1M events/minute peak, <1 minute detection latency
- Dead letter queue: handle malformed/unprocessable events

**Staff-level differentiators**:
- Event ordering guarantees (what if events arrive out of order?)
- Exactly-once processing semantics (checkpointing strategy)
- Back-pressure handling (what if processing can't keep up?)
- Schema evolution (how to add new event types without breaking consumers)

---

### Q4: Design an AI-Powered ERP Integration Layer
**Prompt**: "Our customers use different ERP systems (SAP, Oracle, NetSuite). Design a system that connects to any ERP and normalizes the data for our AI platform."

**Key points to hit**:
- Anti-corruption layer (translate ERP-specific schemas to canonical model)
- Connector architecture (plugin-based: one connector per ERP type)
- Data extraction: CDC vs API polling vs file-based (trade-offs)
- Schema mapping: AI-assisted mapping from ERP fields to canonical model
- Incremental sync (only new/changed data, not full dumps)
- Error handling: what if ERP is down? Data quality issues?
- Security: credentials management, VPN/private link to on-prem ERPs
- Multi-tenant: shared infrastructure, isolated data

**Staff-level differentiators**:
- "AI-assisted onboarding": use LLM to auto-suggest field mappings
- Discuss the long tail: major ERPs have known schemas, but every instance is customized
- Data validation: detect and flag when ERP data doesn't match expected patterns
- Propose a "connector SDK" so new ERP integrations can be built by the team rapidly

---

## TIER 2: Classic Google-Style (Still Very Possible)

### Q5: Design a Notification System
**Prompt**: "Design a notification system that sends 1B notifications per day across push, email, and SMS."

**Key points to hit**:
- API for notification creation (template, recipient, channel, priority)
- Rate limiting per user (don't spam)
- Priority queues (critical alerts before marketing)
- Channel routing (user preferences, fallback strategy)
- Template engine (personalization, localization)
- Delivery tracking (sent, delivered, read, failed)
- Scale: 1B/day = 12K QPS, but bursty (100K QPS peak)
- Retry strategy per channel (email retry longer than push)

---

### Q6: Design a Rate Limiter
**Prompt**: "Design a distributed rate limiter for an API gateway handling 100K QPS."

**Key points to hit**:
- Algorithms: token bucket (API rate limiting), sliding window (more accurate)
- Distributed: Redis-based with Lua scripts for atomicity
- Consistency: slightly over-counting OK vs under-counting (fail open vs closed)
- Multi-dimension: per-user, per-IP, per-endpoint, global
- Response: HTTP 429 with Retry-After header
- Race conditions: what if two nodes both check simultaneously?
- Performance: Must add <1ms latency (Redis is fine)

---

### Q7: Design a Distributed Task Scheduler
**Prompt**: "Design a system that executes millions of scheduled tasks per day with at-least-once execution guarantee."

**Key points to hit**:
- Task persistence (durable storage: PostgreSQL or DynamoDB)
- Time-based partitioning (upcoming tasks hot, old tasks cold)
- Worker fleet (pull-based with lease/heartbeat mechanism)
- Exactly-once vs at-least-once (usually at-least-once with idempotent tasks)
- Failure handling: missed execution window, retry, dead-letter
- Scale: partition by scheduled time + hash for distribution
- Cron-style recurring tasks: expand to individual executions
- Visibility: task status API, execution history

---

### Q8: Design a Web Crawler
**Prompt**: "Design a web crawler that indexes 1B pages with freshness requirements."

**Key points to hit**:
- URL frontier (priority queue: important pages crawled more frequently)
- Politeness: respect robots.txt, rate limit per domain
- Deduplication: URL normalization, content fingerprinting (simhash)
- Architecture: distributed workers, shared URL frontier
- Storage: raw pages → S3, metadata → distributed DB, links → graph DB
- Freshness: re-crawl frequency based on page change rate
- Scale: 1B pages, 10KB avg = 10 PB. Incremental, not all at once.

---

## TIER 3: Moonshot/Novel Problems (X-Specific)

### Q9: Design a System to Predict Supply Chain Disruptions
**Prompt**: "Design a system that predicts supply chain disruptions 2 weeks before they happen."

**Approach hints**:
- Multi-source signal collection (news, weather, port data, financial markets, social media)
- NLP pipeline for news analysis (entity extraction, sentiment, event detection)
- Risk scoring model (per supplier, per route, per region)
- Graph-based propagation (if tier-2 supplier disrupted, which tier-1 affected?)
- Uncertainty quantification (probability + severity + confidence)
- Action recommendations (alternative suppliers, buffer stock, re-routing)
- Feedback loop: did predicted disruption materialize?

---

### Q10: Design a System That Automatically Maps a Company's Supply Chain
**Prompt**: "Given access to a company's ERP database and documents, design a system that automatically discovers and maps their entire supply chain network."

**Approach hints**:
- Data sources: purchase orders, invoices, shipping records, contracts
- Entity extraction: suppliers, warehouses, products, routes
- Relationship inference: who supplies what to whom, through what path
- Graph construction: nodes (entities) + edges (flows)
- Validation: present discovered topology to domain experts for confirmation
- Incremental: learn from corrections, update as new data flows in
- Visualization: interactive graph UI for exploration
- Anomaly: detect unexpected paths (potential fraud, non-compliance)

---

### Q11: Design a "What-If" Simulation Engine
**Prompt**: "Design a system where supply chain managers can ask 'what if supplier X shuts down?' or 'what if demand doubles next month?' and see the impact across their entire supply chain."

**Approach hints**:
- Digital twin: model of current supply chain state
- Simulation engine: discrete event simulation or system dynamics
- Scenario API: parameterized perturbations (remove node, scale demand, add lead time)
- Compute: simulations may take seconds to minutes → async with caching
- Visualization: before/after comparison, cascade visualization
- Recommendations: "here's how to mitigate this scenario"
- Scale: run 1000 Monte Carlo scenarios for probabilistic impact

---

## Self-Evaluation Rubric

After each practice session, score yourself:

| Criterion | 1 (Weak) | 3 (Adequate) | 5 (Strong) |
|-----------|----------|--------------|-------------|
| Requirements | Didn't clarify | Asked basics | Proactively scoped, quantified |
| Structure | Rambling | Organized | Clear phases, signposted transitions |
| Depth | Surface-level | Reasonable detail | Deep on hard problems, quantified trade-offs |
| Breadth | Missing components | All major components | Cross-cutting concerns (security, ops, cost) |
| Trade-offs | None discussed | Mentioned | Quantified, compared alternatives with reasoning |
| Communication | Confusing | Clear | Collaborative, invited discussion, structured |
| Staff signals | Junior-level | Senior-level | Drove direction, identified risks, thought org-wide |
| AI/ML knowledge | Generic | Solid fundamentals | Production-grade depth, aware of pitfalls |

**Target**: Score 4+ on all criteria. Focus your remaining study time on any criterion scoring <3.

---

## Practice Schedule (Fits Your 7-Day Plan)

| Day | Practice |
|-----|----------|
| Day 1 | Read frameworks, sketch Q6 (rate limiter) as warm-up |
| Day 2 | Full 45-min: Q5 (notification system) |
| Day 3 | Full 45-min: Q1 (enterprise knowledge base) — CRITICAL |
| Day 4 | Full 45-min: Q2 (demand forecasting) — CRITICAL |
| Day 5 | Full 45-min: Q3 (event processing) + Q4 (ERP integration) |
| Day 6 | Full 45-min: Q9 or Q10 (moonshot problem) |
| Day 7 | Review weak areas, re-do one problem cleanly |
