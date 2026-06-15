# AI/ML Systems Design (Critical for X Role)

## RAG System Architecture

### What is RAG?
Retrieval-Augmented Generation: Enhance LLM responses with relevant context retrieved from a knowledge base. Critical for enterprise AI where the LLM needs access to private, up-to-date data.

### Full RAG Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                     INGESTION PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│ Source Docs → Chunking → Embedding → Vector DB              │
│ (PDFs, ERP,   (semantic/   (text-embedding-   (Pinecone,    │
│  Confluence,   fixed-size/   3-large,           Weaviate,    │
│  Slack)        recursive)    Cohere embed)      pgvector)    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     QUERY PIPELINE                            │
├─────────────────────────────────────────────────────────────┤
│ User Query → Query Transform → Retrieval → Reranking → LLM │
│              (expansion,       (vector +    (cross-encoder,  │
│               HyDE,            keyword      Cohere rerank)   │
│               decomposition)   hybrid)                       │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

#### Chunking Strategy
| Strategy | Chunk Size | Best For |
|----------|-----------|----------|
| Fixed-size | 256-512 tokens | Simple, predictable |
| Recursive/semantic | Variable | Documents with structure |
| Sentence-level | 1-3 sentences | FAQ, definitions |
| Document-level | Full doc | Short documents, metadata-rich |
| Sliding window | Overlap 20% | Context continuity |

**Staff insight**: Chunking is the #1 factor in RAG quality. Too small = lost context. Too large = diluted relevance. Always include surrounding context (parent-child chunking).

#### Retrieval Strategies
- **Dense retrieval**: Embedding similarity (cosine/dot product). Good for semantic meaning.
- **Sparse retrieval**: BM25/TF-IDF. Good for exact keyword matches.
- **Hybrid**: Combine both with Reciprocal Rank Fusion (RRF). Best in practice.
- **Multi-vector**: ColBERT-style token-level matching. Higher quality, more compute.

```python
# Hybrid search with RRF
def hybrid_search(query, k=10):
    dense_results = vector_db.search(embed(query), top_k=k*2)
    sparse_results = elasticsearch.search(query, top_k=k*2)
    return reciprocal_rank_fusion(dense_results, sparse_results, k=k)

def reciprocal_rank_fusion(results_lists, k=60):
    scores = defaultdict(float)
    for results in results_lists:
        for rank, doc in enumerate(results):
            scores[doc.id] += 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])
```

#### Reranking
After initial retrieval (cheap, broad), apply expensive cross-encoder reranking:
```
Retrieved 50 docs (fast, vector search)
    → Rerank with cross-encoder → Top 5 (high quality, expensive)
        → Feed to LLM context window
```

#### Advanced RAG Patterns
- **Query decomposition**: Break complex queries into sub-queries, retrieve for each
- **HyDE (Hypothetical Document Embeddings)**: Generate hypothetical answer, embed that for retrieval
- **Self-RAG**: LLM decides when to retrieve and self-grades relevance
- **Corrective RAG (CRAG)**: If retrieval quality is low, fall back to web search or different strategy
- **Agentic RAG**: LLM uses tools to iteratively retrieve and reason

---

## LLM Serving Architecture

### Production LLM System
```
┌──────────────────────────────────────────────────────┐
│                    API GATEWAY                         │
│  Rate limiting, auth, request routing, cost tracking  │
├──────────────────────────────────────────────────────┤
│                  ORCHESTRATION LAYER                   │
│  Prompt management, chain execution, tool routing     │
├──────────────────────────────────────────────────────┤
│              ┌──────────┐  ┌───────────┐             │
│              │ LLM Pool │  │ Guard-    │             │
│              │ (routing) │  │ rails     │             │
│              └──────────┘  └───────────┘             │
│                   │                │                   │
│         ┌─────────┼────────────────┤                  │
│         │         │                │                   │
│    ┌────▼───┐ ┌───▼────┐ ┌───────▼──────┐           │
│    │GPT-4/  │ │Claude  │ │Fine-tuned    │           │
│    │Gemini  │ │        │ │local model   │           │
│    └────────┘ └────────┘ └──────────────┘           │
├──────────────────────────────────────────────────────┤
│              EVALUATION & OBSERVABILITY                │
│  Tracing, quality metrics, cost tracking, A/B tests   │
└──────────────────────────────────────────────────────┘
```

### Key Components

#### Prompt Management
```python
# Version-controlled prompts with A/B testing
class PromptRegistry:
    def get_prompt(self, name, version=None, context=None):
        template = self.store.get(name, version or "latest")
        # Dynamic prompt selection based on context
        if context.complexity == "high":
            return template.variants["detailed"]
        return template.variants["concise"]
```

#### Guardrails
- **Input guardrails**: PII detection, prompt injection detection, topic filtering
- **Output guardrails**: Hallucination detection, format validation, toxicity checks
- **Structural guardrails**: Output schema enforcement (JSON mode, function calling)

#### Model Router
Route to appropriate model based on:
- Task complexity (simple → Haiku, complex → Opus)
- Latency requirements (streaming vs batch)
- Cost constraints
- Quality requirements

```python
def route_request(request):
    if request.requires_reasoning:
        return "claude-opus-4"
    elif request.high_throughput:
        return "claude-haiku-4"
    elif request.domain_specific:
        return "fine_tuned_model_v3"
    return "claude-sonnet-4"
```

#### Caching for LLMs
- **Semantic caching**: Embed the query, check if similar query was recently answered
- **Exact caching**: Hash of (prompt + params) → cached response
- **Prefix caching**: Cache KV states for shared prompt prefixes (Anthropic supports this)

---

## Embedding Pipeline Design

### Architecture
```
Raw Content → Preprocessing → Chunking → Embedding Model → Vector DB
                  │                           │                  │
            (cleaning,              (batch inference,     (HNSW index,
             normalization,          GPU cluster,          metadata
             language detection)     rate limiting)        filtering)
```

### Scaling Considerations
- **Batch vs real-time**: Batch for bulk ingestion, real-time for new documents
- **Model selection**: text-embedding-3-large (OpenAI), Cohere embed-v3, open-source (BGE, E5)
- **Dimensionality**: 768-3072 dimensions. More = better quality, more storage/compute
- **Quantization**: Reduce dimensions or quantize (int8) for 4x storage savings with <5% quality loss

### Vector Database Selection
| Database | Type | Best For |
|----------|------|----------|
| Pinecone | Managed | Zero-ops, auto-scaling |
| Weaviate | Open-source | Hybrid search, multi-modal |
| Qdrant | Open-source | Filtering + vector search |
| pgvector | Extension | Already on PostgreSQL, moderate scale |
| Milvus | Open-source | Billion-scale, GPU support |
| Chroma | Lightweight | Prototyping, small-medium scale |

### HNSW (Hierarchical Navigable Small World) — How Vector Search Works
- Build navigable graph layers (coarse → fine)
- Search: start at top layer, greedily traverse to nearest neighbors, descend
- Parameters: `ef_construction` (build quality), `M` (connections per node), `ef_search` (query quality)
- Trade-off: Higher params = better recall, more memory, slower build

---

## ML Model Serving

### Serving Patterns
| Pattern | Latency | Throughput | Use Case |
|---------|---------|------------|----------|
| Online (real-time) | <100ms | Low-medium | User-facing predictions |
| Batch | Hours | Very high | Nightly reports, bulk scoring |
| Near-real-time | Seconds | Medium | Streaming predictions |
| Edge | <10ms | Low | IoT, mobile |

### Online Serving Architecture
```
Request → Feature Store (online) → Model Server → Post-processing → Response
              │                        │
         (pre-computed             (TensorRT/
          features, <10ms)          Triton/
                                    vLLM)
```

### Feature Store
**Why**: Consistency between training and serving. Avoid training-serving skew.

```
┌─────────────────────────────────────────┐
│           FEATURE STORE                  │
├──────────────────┬──────────────────────┤
│  Offline Store   │   Online Store        │
│  (training data) │   (serving features)  │
│  Parquet/BQ      │   Redis/DynamoDB      │
│  High latency    │   <10ms latency       │
└──────────────────┴──────────────────────┘
```

**Key features**:
- Point-in-time correctness (no future leakage in training)
- Feature versioning and lineage
- Monitoring for data drift and staleness

---

## LLM Evaluation & Observability

### Evaluation Framework
```
Offline Eval                    Online Eval
────────────                    ───────────
- Benchmark datasets            - A/B tests
- Human preference              - User feedback (👍/👎)
- LLM-as-judge                  - Task completion rate
- RAG metrics (RAGAS):          - Hallucination rate
  - Faithfulness                - Latency percentiles
  - Answer relevancy            - Cost per query
  - Context precision
  - Context recall
```

### Key Metrics for Production LLM Systems
- **Latency**: Time to first token (TTFT), tokens/second, total response time
- **Quality**: Answer accuracy, hallucination rate, format compliance
- **Cost**: $/query, $/token, cache hit rate
- **Reliability**: Error rate, timeout rate, retry rate
- **Safety**: Guardrail trigger rate, PII leak rate

### Observability Stack
```
Application → OpenTelemetry → Traces/Metrics/Logs
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
              Trace Store      Metric Store     Log Store
              (Jaeger)         (Prometheus)     (Loki)
                    │               │               │
                    └───────────────┼───────────────┘
                                    │
                              Grafana Dashboards
                              + PagerDuty Alerts
```

---

## AI Agent Architecture

### Multi-Agent Systems (Relevant for supply chain)
```
┌────────────────────────────────────────┐
│           ORCHESTRATOR AGENT            │
│  (decomposes task, routes to specialists)│
├────────────┬──────────┬────────────────┤
│ Demand     │ Inventory│ Logistics      │
│ Forecaster │ Optimizer│ Router         │
│ Agent      │ Agent    │ Agent          │
├────────────┼──────────┼────────────────┤
│ Tools:     │ Tools:   │ Tools:         │
│ - SQL query│ - ERP API│ - Maps API     │
│ - Stats    │ - Cost   │ - Carrier APIs │
│ - Charts   │   calc   │ - Route solver │
└────────────┴──────────┴────────────────┘
```

### Agent Design Patterns
- **ReAct**: Reason → Act → Observe → Repeat
- **Plan-and-Execute**: Plan all steps first, then execute sequentially
- **Reflexion**: Self-evaluate after execution, retry with learnings
- **Multi-agent debate**: Multiple agents argue, consensus emerges

### Tool Use Architecture
```python
tools = [
    Tool(name="query_inventory", fn=query_inventory_db,
         description="Get current inventory levels for SKUs"),
    Tool(name="get_demand_forecast", fn=call_forecast_model,
         description="Get demand prediction for next N days"),
    Tool(name="calculate_reorder_point", fn=reorder_calc,
         description="Calculate optimal reorder point given lead time and demand"),
]
```

---

## Handling Hallucination in Enterprise Systems

### Prevention Strategies
1. **Grounding**: Always provide source documents; instruct model to cite
2. **Constrained generation**: JSON mode, function calling, enum responses
3. **Retrieval verification**: Check if answer is supported by retrieved context
4. **Confidence scoring**: Model self-reports confidence; low confidence → human review
5. **Chain-of-verification**: Generate answer → generate verification questions → check consistency

### Detection Methods
- **NLI-based**: Use natural language inference model to check if response is entailed by context
- **LLM-as-judge**: Second LLM verifies claims against source
- **Factual consistency scores**: Compare entities/claims between source and response
- **Citation verification**: Check that cited passages actually support the claim

### Staff Insight: Enterprise AI Reliability
For supply chain decisions (where hallucinations cost real money):
- Never let LLM make autonomous decisions above $X threshold
- Always show provenance (which documents informed this answer)
- Human-in-the-loop for high-stakes actions
- Audit trail for regulatory compliance
- Graceful degradation: if AI is uncertain, fall back to rule-based system
