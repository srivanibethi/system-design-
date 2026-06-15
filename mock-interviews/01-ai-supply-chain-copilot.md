# Mock Interview 1: AI-Powered Supply Chain Copilot

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "We want to build an AI assistant for supply chain managers. They should be able to ask natural language questions about their operations — things like 'Why did we stockout last week?', 'What's our supplier risk exposure?', or 'Recommend optimal reorder quantities for next month.' Design this system end-to-end."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Users & Scale:**
1. "Who are the primary users — supply chain analysts, VPs, warehouse managers? Their technical sophistication affects UX complexity."
2. "How many enterprise customers do we support? Are we multi-tenant?"
3. "What's the expected concurrent user count and query volume?"
4. "Is this a greenfield product or are we augmenting an existing tool?"

**About Data Sources:**
5. "What systems hold the data — SAP, Oracle, custom ERPs? Do we have direct access or go through APIs?"
6. "Is the data structured (SQL tables), unstructured (documents), or both?"
7. "How fresh does the data need to be — real-time or is daily refresh acceptable?"
8. "How much historical data are we talking about? Years? Terabytes?"

**About AI Behavior:**
9. "What's the accuracy bar? Enterprise users making financial decisions need high confidence."
10. "Should the system ever take actions autonomously, or always recommend?"
11. "Do we need to handle multi-turn conversations (follow-up questions)?"
12. "What languages do we need to support?"

**About Constraints:**
13. "Are there compliance requirements — SOC 2, GDPR, data residency?"
14. "What's the latency target for responses — is 10-15 seconds acceptable for complex queries?"
15. "Budget constraints on LLM API costs?"

### Interviewer's Likely Answers:
- Multi-tenant SaaS, 50-100 enterprise customers
- Each has 10-50 concurrent users
- Mix of structured (ERP data) and unstructured (contracts, SOPs)
- Daily data refresh minimum, real-time for critical metrics
- Must cite sources, high accuracy, recommend only (no auto-actions)
- SOC 2 required, US data residency
- 10-15 seconds acceptable for complex queries

---

## PHASE 2: Requirements Summary & Back-of-Envelope (3 minutes)

### State This Explicitly:

```
"Let me summarize what I'm hearing and do some quick math:

FUNCTIONAL REQUIREMENTS:
1. Natural language Q&A grounded in customer's supply chain data
2. Three query types: factual (lookup), analytical (aggregation/trends), 
   recommendation (optimization-backed)
3. Multi-turn conversation with context retention
4. Cited sources for every answer
5. Audit trail for compliance

NON-FUNCTIONAL:
- 100 customers × 50 users × ~5 queries/hour = ~25K queries/hour = ~7 QPS avg
- Peak: 10x = 70 QPS  
- Latency: <5s for simple lookups, <15s for complex analysis
- Availability: 99.9% (44 min downtime/month)
- Data isolation: Tenant A must NEVER see Tenant B's data

COST ESTIMATE:
- 25K queries/hour × $0.03/query (LLM) = $750/hour = $18K/day
- With 60% cache hit rate: $7.2K/day
- This is acceptable for enterprise SaaS ($50K+/year per customer)

Does this scope feel right to you?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Draw This Diagram:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │
│  │ Web App (React)│  │ Chat Widget  │  │ API (for programmatic access)│  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │
└─────────┼──────────────────┼─────────────────────────┼──────────────────┘
          │                  │                         │
          ▼                  ▼                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                                        │
│  Auth (JWT) │ Rate Limiting │ Tenant Routing │ Request Logging           │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION SERVICE                                  │
│                                                                          │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────────────┐ │
│  │ Session Mgr  │    │ Query Classifier  │    │ Response Synthesizer  │ │
│  │ (context,    │    │ (route to right   │    │ (format, cite,       │ │
│  │  history)    │    │  pipeline)        │    │  confidence score)   │ │
│  └──────────────┘    └────────┬─────────┘    └───────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐
│ RAG PIPELINE │ │ ANALYTICS    │ │ RECOMMENDATION       │
│              │ │ PIPELINE     │ │ PIPELINE             │
│ Vector DB +  │ │ Text-to-SQL  │ │ Optimization Engine  │
│ Doc Store    │ │ + Execution  │ │ + ML Models          │
│ + Reranking  │ │ + Viz        │ │ + Constraints Solver │
└──────────────┘ └──────────────┘ └──────────────────────┘
        │               │                    │
        └───────────────┼────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       DATA LAYER (Per Tenant)                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐   │
│  │ Vector DB  │  │ Analytics  │  │ Feature    │  │ Document       │   │
│  │ (embeddings│  │ Warehouse  │  │ Store      │  │ Store (S3)     │   │
│  │  per tenant│  │ (BigQuery) │  │ (Redis)    │  │                │   │
│  └────────────┘  └────────────┘  └────────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
        │                                     ▲
        │                                     │
┌───────┴─────────────────────────────────────┼───────────────────────────┐
│                   INGESTION LAYER            │                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  ┌───────────────┐       │
│  │ ERP CDC  │  │ Doc      │  │ API      │  │  │ Data Quality  │       │
│  │ Connector│  │ Processor│  │ Poller   │  │  │ Pipeline      │       │
│  └──────────┘  └──────────┘  └──────────┘  │  └───────────────┘       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Explain the Flow:

```
"Here's the happy path for a query like 'Why did we stockout on SKU-789?':

1. User types question in chat UI
2. API Gateway authenticates, identifies tenant, rate-checks
3. Orchestration Service:
   a. Session Manager loads conversation history
   b. Query Classifier determines: this is ANALYTICAL (needs data lookup + reasoning)
   c. Routes to BOTH RAG pipeline (for context/documents) AND Analytics pipeline (for data)
4. Analytics Pipeline:
   - Translates to SQL: SELECT inventory_levels, demand, supply_orders WHERE sku='789' AND date=last_week
   - Executes against tenant's warehouse
   - Returns structured data
5. RAG Pipeline:
   - Retrieves relevant documents (reorder policies, supplier SLAs)
   - Provides contextual background
6. Response Synthesizer:
   - Combines data + context + conversation history
   - Generates explanation with citations
   - Assigns confidence score
7. Returns to user with sources cited

The key insight: we don't treat every query the same. The classifier 
routes to specialized pipelines, and the synthesizer combines their outputs."
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Query Classification & Routing

**Interviewer might ask**: "How does the classifier work? What if it misroutes?"

```
"The Query Classifier is a critical component. Here's how I'd design it:

APPROACH: Fine-tuned small model (not LLM — too slow/expensive for routing)

┌─────────────────────────────────────────────────┐
│           QUERY CLASSIFICATION                   │
├─────────────────────────────────────────────────┤
│ Input: "Why did we stockout on SKU-789?"         │
│                                                  │
│ Step 1: Intent Detection (fine-tuned BERT)       │
│   → intent: ROOT_CAUSE_ANALYSIS                  │
│   → confidence: 0.94                             │
│                                                  │
│ Step 2: Entity Extraction                        │
│   → entities: {sku: "SKU-789", event: "stockout",│
│                time: "last_week"}                 │
│                                                  │
│ Step 3: Pipeline Routing                         │
│   → primary: ANALYTICS (needs data)              │
│   → secondary: RAG (needs context/policies)      │
│   → exclude: RECOMMENDATION (not asking for advice)│
│                                                  │
│ Step 4: Confidence Check                         │
│   → if confidence < 0.7: ask clarifying question │
│   → if ambiguous: route to multiple pipelines    │
└─────────────────────────────────────────────────┘

CATEGORIES:
- FACTUAL: "What's the lead time for supplier X?" → RAG
- ANALYTICAL: "What were our top 10 stockouts last month?" → Text-to-SQL
- DIAGNOSTIC: "Why did X happen?" → Analytics + RAG (combined)
- PREDICTIVE: "Will we stockout next week?" → ML Model Serving
- RECOMMENDATION: "What should I reorder?" → Optimization Engine
- CONVERSATIONAL: "Can you explain that further?" → Context from session

MISROUTING MITIGATION:
1. Low-confidence queries → fan out to multiple pipelines, synthesize
2. Feedback loop: user 👎 triggers reclassification analysis
3. Fallback: if specialized pipeline fails, fall back to general RAG
4. A/B test routing decisions, measure satisfaction per route

The classifier adds <50ms latency (it's a small model). The cost of 
misrouting is higher latency (ran wrong pipeline first), not incorrect 
answers (synthesizer validates before responding)."
```

---

### Deep Dive 2: Multi-Tenant Data Isolation

**Interviewer might ask**: "How do you ensure tenant A never sees tenant B's data?"

```
"This is the #1 security requirement. I'd use defense-in-depth:

LAYER 1: Authentication & Context
┌────────────────────────────────────┐
│ Every request carries tenant_id     │
│ Extracted from JWT at API gateway   │
│ Propagated via request context      │
│ (like a thread-local / context var) │
└────────────────────────────────────┘

LAYER 2: Data Isolation Model
┌────────────────────────────────────────────────────────┐
│ OPTION A: Shared DB, tenant_id column                   │
│   Pro: Simpler ops, cheaper                             │
│   Con: Risk of query without WHERE tenant_id=X          │
│                                                         │
│ OPTION B: Separate schemas per tenant                   │
│   Pro: Stronger isolation, easy to reason about         │
│   Con: More operational complexity                      │
│                                                         │
│ OPTION C: Separate databases per tenant                 │
│   Pro: Strongest isolation, independent scaling         │
│   Con: Expensive, hard to manage at 100+ tenants        │
│                                                         │
│ MY CHOICE: Option B (separate schemas) with             │
│ per-tenant encryption keys for data at rest.            │
│ Reason: Strong isolation without ops explosion.         │
│ We can shard large tenants to their own DB later.       │
└────────────────────────────────────────────────────────┘

LAYER 3: Vector DB Isolation
- Namespace per tenant in vector DB (e.g., Pinecone namespaces)
- Every vector search MUST include tenant namespace filter
- Embedding models are shared (stateless), but stored embeddings are isolated

LAYER 4: LLM Prompt Isolation
- Tenant data is injected into context window per-request
- No fine-tuning on multi-tenant data (avoids memorization leakage)
- Prompt includes: "Only answer based on the provided context. 
  Do not reference any information not in the context."

LAYER 5: Automated Verification
- Integration tests that attempt cross-tenant access (must fail)
- Audit logging: every data access logged with tenant context
- Quarterly penetration testing

The paranoia level here is appropriate because a data leak between 
enterprise customers is an existential business risk."
```

---

### Deep Dive 3: RAG Pipeline Quality

**Interviewer might ask**: "How do you ensure high-quality answers? What about hallucination?"

```
"RAG quality is a pipeline problem with multiple stages to optimize:

┌─────────────────────────────────────────────────────────────┐
│               RAG QUALITY PIPELINE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAGE 1: RETRIEVAL QUALITY                                 │
│  ─────────────────────────                                  │
│  Problem: Retrieve the RIGHT documents                       │
│                                                              │
│  Query: "Why did we stockout on SKU-789?"                   │
│         ↓                                                    │
│  Query Expansion: "stockout SKU-789 inventory depletion      │
│                    out-of-stock shortage"                     │
│         ↓                                                    │
│  Hybrid Search:                                              │
│    Dense (semantic): Embed query → vector search (top 20)    │
│    Sparse (keyword): BM25 search for "SKU-789" (top 20)     │
│    Metadata filter: tenant_id=X, doc_type IN (inventory,     │
│                     reorder_policy, supplier_info)            │
│         ↓                                                    │
│  Reciprocal Rank Fusion → Merged top 20                      │
│         ↓                                                    │
│  Cross-Encoder Reranking → Top 5                             │
│                                                              │
│  METRICS: Recall@5, MRR, NDCG                               │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAGE 2: GENERATION QUALITY                                 │
│  ───────────────────────────                                 │
│  Problem: Generate FAITHFUL answer from retrieved context     │
│                                                              │
│  Prompt Structure:                                           │
│  ┌──────────────────────────────────────────────────┐       │
│  │ SYSTEM: You are a supply chain analyst assistant.  │       │
│  │ Answer ONLY from the provided context.             │       │
│  │ If the context doesn't contain the answer,         │       │
│  │ say "I don't have enough information."             │       │
│  │ Always cite which source supports each claim.      │       │
│  │                                                    │       │
│  │ CONTEXT: [5 retrieved chunks with source IDs]      │       │
│  │                                                    │       │
│  │ CONVERSATION HISTORY: [prior turns]                │       │
│  │                                                    │       │
│  │ USER: Why did we stockout on SKU-789?              │       │
│  └──────────────────────────────────────────────────┘       │
│                                                              │
│  Output Schema (enforced via function calling):              │
│  {                                                           │
│    "answer": "...",                                          │
│    "citations": [{"source_id": "doc_123", "quote": "..."}], │
│    "confidence": 0.85,                                       │
│    "reasoning": "Based on inventory data showing..."         │
│  }                                                           │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAGE 3: VERIFICATION                                       │
│  ─────────────────────                                       │
│  Problem: Catch hallucinations before reaching user          │
│                                                              │
│  Check 1: Citation verification                              │
│    - For each citation, verify the quote exists in source    │
│    - If any citation is fabricated → regenerate or warn      │
│                                                              │
│  Check 2: Confidence threshold                               │
│    - confidence < 0.6 → "I'm not confident about this.      │
│      Here's what I found, but please verify: ..."            │
│    - confidence < 0.3 → "I don't have enough information     │
│      to answer this reliably."                               │
│                                                              │
│  Check 3: Consistency check (for critical queries)           │
│    - Generate answer twice with different retrieved docs      │
│    - If answers contradict → flag for human review           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

EVALUATION FRAMEWORK:
┌──────────────────────────────────────────────────┐
│ OFFLINE (weekly):                                 │
│ - 200 golden Q&A pairs per customer segment       │
│ - Measure: faithfulness, relevancy, completeness  │
│ - Automated via LLM-as-judge (RAGAS framework)    │
│                                                   │
│ ONLINE (continuous):                              │
│ - User feedback: 👍/👎 on every answer            │
│ - Click-through on citations (did user verify?)   │
│ - Session length (longer = engaged OR confused)   │
│ - Escalation rate (asked a human after AI answer) │
│                                                   │
│ ALERTING:                                         │
│ - Satisfaction drops below 80% → page oncall      │
│ - Hallucination detected → immediate investigation│
│ - New topic cluster with low confidence → gap in  │
│   knowledge base                                  │
└──────────────────────────────────────────────────┘
```

---

## PHASE 5: Operational Concerns & Wrap-Up (5 minutes)

### Monitoring & Observability
```
"For production readiness, I'd instrument:

KEY METRICS:
- Query latency percentiles: p50, p95, p99 (target: <5s/<15s/<30s)
- Answer quality: daily avg confidence score, 👍 rate
- LLM cost per query: track by query type and customer tier
- Retrieval metrics: avg docs retrieved, reranking changes, cache hit rate
- Error rate: LLM timeouts, retrieval failures, empty results

TRACING (OpenTelemetry):
User query → classify (50ms) → retrieve (200ms) → rerank (100ms) → 
  generate (3-8s) → verify (500ms) → respond

Every trace includes: tenant_id, query_type, model_used, tokens_consumed,
                      docs_retrieved, confidence_score, user_feedback

ALERTING:
- p99 latency > 30s for 5 minutes → page oncall
- Confidence score avg drops 10% day-over-day → investigate
- LLM provider error rate > 5% → activate fallback provider
"
```

### Deployment Strategy
```
"I'd deploy incrementally:

PHASE 1 (Month 1): 3 design partners, RAG-only (factual Q&A)
  - Learn: What questions do they actually ask?
  - Validate: Is retrieval quality sufficient?

PHASE 2 (Month 2-3): Add analytics pipeline (Text-to-SQL)
  - Learn: How complex are their analytical needs?
  - Validate: SQL accuracy + safety

PHASE 3 (Month 4-6): Add recommendations, scale to 20 customers
  - Learn: Do they trust AI recommendations?
  - Validate: Recommendation quality vs current process

Canary deployment: New versions get 5% traffic, monitor for quality 
regression, promote to 100% after 24h of green metrics."
```

### Extensions (What V2 Looks Like)
```
"If we had more time, I'd explore:

1. PROACTIVE INSIGHTS: System detects anomalies and pushes alerts
   ('I noticed demand for SKU-789 spiked 3x today — want me to 
   investigate?')

2. COLLABORATIVE FEATURES: Share an AI finding with a colleague,
   they can ask follow-up questions with full context

3. LEARNING FROM ACTIONS: Track which recommendations users act on,
   use this as reward signal for improving recommendations

4. MULTI-MODAL: Accept photos (damaged goods, warehouse layouts),
   generate charts/visualizations in responses

5. AUTONOMOUS AGENTS: For low-risk decisions below $X threshold,
   allow the system to auto-execute (e.g., reorder standard supplies)"
```

---

## Scoring Rubric (Self-Evaluate)

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Scoping | Quantified scale, identified 3 query types, stated explicit assumptions | ☐ |
| Architecture | Clean separation of concerns, right components for the problem | ☐ |
| AI Depth | RAG pipeline details, hallucination mitigation, evaluation framework | ☐ |
| Multi-tenancy | Defense-in-depth isolation, not just "add tenant_id column" | ☐ |
| Trade-offs | Quantified cost, compared approaches, stated reasoning | ☐ |
| Operations | Monitoring, deployment strategy, failure modes | ☐ |
| Staff signals | Drove conversation, proposed phased rollout, org implications | ☐ |
| Communication | Structured, clear, invited discussion | ☐ |
