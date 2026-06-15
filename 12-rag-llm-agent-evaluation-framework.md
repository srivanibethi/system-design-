# RAG, LLM & Agent Evaluation Framework

## Complete Guide to Testing Quality, Metrics, and Production Evaluation

---

## Table of Contents

1. [RAG Evaluation Frameworks](#1-rag-evaluation-frameworks)
2. [Retrieval Quality Metrics](#2-retrieval-quality-metrics)
3. [Generation Quality Metrics](#3-generation-quality-metrics)
4. [LLM Evaluation Benchmarks](#4-llm-evaluation-benchmarks)
5. [LLM-as-Judge Methodology](#5-llm-as-judge-methodology)
6. [Agent Evaluation](#6-agent-evaluation)
7. [Tool Calling Evaluation](#7-tool-calling-evaluation)
8. [Multi-Step Reasoning Evaluation](#8-multi-step-reasoning-evaluation)
9. [Production Monitoring & Observability](#9-production-monitoring--observability)
10. [Open-Source Judge Models](#10-open-source-judge-models)
11. [Cost & Latency Evaluation](#11-cost--latency-evaluation)
12. [Google-Scale Evaluation Architecture](#12-google-scale-evaluation-architecture)
13. [Interview Discussion Framework](#13-interview-discussion-framework)

---

## 1. RAG Evaluation Frameworks

### 1.1 RAGAS (Retrieval Augmented Generation Assessment)

**Repository:** github.com/explodinggradients/ragas | **Stars:** 14.4k | **License:** Apache-2.0

#### Core Metrics

| Metric | Formula | What It Measures |
|--------|---------|-----------------|
| **Faithfulness** | `Supported claims / Total claims` | Are answers grounded in retrieved context? |
| **Answer Relevancy** | `(1/N) * SUM(cos_sim(E_gen_q_i, E_original))` | Does the answer address the question? |
| **Context Precision** | `SUM(Precision@k * v_k) / Total relevant in K` | Are relevant chunks ranked higher? |
| **Context Recall** | `Reference claims supported by context / Total reference claims` | Does retrieval capture all needed info? |

#### Faithfulness (Deep Dive)
```
Step 1: LLM extracts all claims from the response
Step 2: For each claim, verify if context supports it
Score = Number of supported claims / Total claims
```
- 2 LLM calls per evaluation
- Variant: `FaithfulnesswithHHEM` uses Vectara's HHEM-2.1-Open T5 classifier (no LLM needed)

#### Answer Relevancy (Deep Dive)
```
Step 1: LLM generates N artificial questions from the response
Step 2: Embed each generated question + original question
Step 3: Compute average cosine similarity
Score = mean(cos_sim(gen_q_i, original_q))
```
- Hybrid: LLM for question generation + embeddings for similarity
- Captures whether the answer contains extraneous information

#### Context Precision (Deep Dive)
```
Score = SUM(Precision@k * v_k) / Total relevant items in top K
Where v_k = 1 if item at rank k is relevant, 0 otherwise
```
- Weighted cumulative precision rewarding correct ranking
- Variants: `LLMContextPrecisionWithReference`, `WithoutReference`, `NonLLM` (Levenshtein)

#### Additional RAGAS Metrics (30 total)
- Context Entities Recall, Noise Sensitivity
- Factual Correctness (F1/precision/recall)
- Semantic Similarity, BLEU/ROUGE
- Tool Call Accuracy/F1, Agent Goal Accuracy
- Topic Adherence, Aspect Critic, Rubrics-based Scoring

---

### 1.2 DeepEval

**Stars:** 16.2k | **License:** Apache-2.0 | **Metrics:** 50+

#### Core RAG Metrics

| Metric | Formula |
|--------|---------|
| **Faithfulness** | `Truthful Claims / Total Claims` (truthful = not contradicting context) |
| **Answer Relevancy** | `Relevant Statements / Total Statements` (reference-free) |
| **Contextual Precision** | `(1/Relevant Nodes) * SUM(Relevant Up To k / k * r_k)` |
| **Contextual Recall** | `Attributable Statements / Total Statements` |
| **Hallucination** | `Contradicted Contexts / Total Contexts` (LOWER = better) |

#### Key Differentiators

| Feature | DeepEval | RAGAS |
|---------|----------|-------|
| Philosophy | Engineering/CI/CD | Research/academic |
| CI/CD | Pytest-native (`assert_test`) | Not primary |
| Custom metrics | GEval + DAG (deterministic) | Limited |
| Caching/cost | Built-in | None |
| Best for | Dev testing pipelines | RAG benchmarking |

#### GEval Metric (Custom LLM-as-Judge)
- Chain-of-thought reasoning before scoring
- Token probability normalization for continuous scores
- Configurable evaluation steps

#### DAG Metric (Deterministic Evaluation)
- Decision-tree evaluation with 4 node types
- Reproducible path-based scoring
- No LLM variance

---

### 1.3 TruLens

**Stars:** 3.4k | **License:** MIT

#### The RAG Triad

| Metric | Method |
|--------|--------|
| **Context Relevance** | Each chunk scored 0-3, normalized. LLM or HuggingFace model. |
| **Groundedness** | Separate response into claims, NLI-based evidence search per claim. |
| **Answer Relevance** | Score 0-3 via `relevance_with_cot_reasons`, normalized. |

- Providers: OpenAI, Anthropic, Cohere, Mistral, HuggingFace, Cortex
- OpenTelemetry-based instrumentation (2025)
- 5.4x speedup with batch evaluation

---

### 1.4 LangSmith

**Type:** Commercial | **Pricing:** $39/seat/month + usage

- Built-in RAG evaluators: Helpfulness, Groundedness, Retrieval Relevance, Correctness, Hallucination
- Evaluator types: LLM-as-Judge, Code-based, Agent trajectory, Summary, Composite
- Modes: Offline (pre-deployment datasets) + Online (production sampling)

---

### 1.5 Arize Phoenix

**Stars:** 12k+ | **License:** ELv2

- OpenTelemetry + OpenInference with 11 span kinds
- UMAP 2D/3D embedding visualization for failure discovery
- HDBSCAN clustering for failure mode analysis
- Euclidean distance drift detection
- IR metrics: NDCG@k, Precision@k, Hit Rate

---

### 1.6 Framework Comparison

| Dimension | RAGAS | DeepEval | TruLens | LangSmith | Phoenix |
|-----------|-------|----------|---------|-----------|---------|
| Primary Focus | RAG metrics | Unit testing | Monitoring | Full lifecycle | Observability |
| RAG Metrics | 8 dedicated | 5 + hallucination | 3 (Triad) | 4 templates | 3 + IR |
| Non-LLM Options | Yes (HHEM) | Limited | Yes (HuggingFace) | Yes (code) | Yes (code) |
| Best For | Benchmarking | CI/CD pipelines | Flexible monitoring | Full-stack ops | Debugging |

---

## 2. Retrieval Quality Metrics

### 2.1 Information Retrieval Metrics

| Metric | Formula | Use Case |
|--------|---------|----------|
| **Precision@k** | `Relevant in top-k / k` | Are retrieved docs relevant? |
| **Recall@k** | `Relevant in top-k / Total relevant` | Are all relevant docs retrieved? |
| **MRR** | `1/N * SUM(1/rank_i)` | How high is first relevant result? |
| **NDCG@k** | `DCG@k / IDCG@k` | Quality-weighted ranking measure |
| **Hit Rate** | `Queries with >= 1 relevant in top-k / Total queries` | Binary retrieval success |
| **MAP** | `Mean of AP across queries` | Overall ranking quality |

### 2.2 DCG/NDCG Calculation

```
DCG@k = SUM(i=1 to k) [ relevance_i / log2(i + 1) ]
IDCG@k = DCG@k of ideal ranking
NDCG@k = DCG@k / IDCG@k
```

### 2.3 Embedding Quality Metrics

| Metric | What It Measures |
|--------|-----------------|
| Cosine Similarity Distribution | How well embeddings separate relevant from irrelevant |
| Embedding Drift (Euclidean) | Has the embedding space shifted over time? |
| Cluster Purity | Do semantic clusters align with relevance labels? |

### 2.4 Chunking Strategy Evaluation

| Strategy | Metric | Trade-off |
|----------|--------|-----------|
| Fixed-size chunks | Recall@k | Simple but may split concepts |
| Semantic chunking | Context Precision | Better coherence, higher compute |
| Hierarchical | NDCG@k | Multi-granularity, complex indexing |
| Sentence-window | Answer Relevancy | Good context, more tokens per chunk |

---

## 3. Generation Quality Metrics

### 3.1 Faithfulness / Groundedness

```
Claim Extraction -> Evidence Matching -> Score

Faithfulness = Claims supported by context / Total claims
```

**Methods:**
- NLI-based (Natural Language Inference): DeBERTa, BART-MNLI
- LLM-based: Extract claims, verify each against context
- HHEM (Hallucination Detection): Vectara's T5-based binary classifier

### 3.2 Hallucination Detection

| Approach | Speed | Accuracy | Cost |
|----------|-------|----------|------|
| SelfCheckGPT (consistency) | Medium | High | N samples |
| NLI classifiers | Fast | Medium | Single pass |
| LLM claim verification | Slow | Highest | 2 LLM calls |
| HHEM-2.1-Open | Fast | Good | Single pass |

**SelfCheckGPT Method:**
```
1. Generate N=20 samples of the response
2. For each claim, check consistency across samples
3. Inconsistency indicates hallucination
4. Scoring: BERTScore, QA, NLI, or LLM binary
```

### 3.3 Answer Quality Metrics

| Metric | Formula/Method | Best For |
|--------|---------------|----------|
| **Semantic Similarity** | `cos_sim(embed(answer), embed(reference))` | When reference exists |
| **ROUGE-L** | Longest common subsequence overlap | Summarization |
| **BLEU** | N-gram precision with brevity penalty | Translation-style tasks |
| **BERTScore** | Token-level embedding similarity (P/R/F1) | Soft semantic matching |
| **Factual Correctness** | Claim-level F1 (precision + recall) | Fact verification |

### 3.4 Answer Relevancy vs Completeness

| Dimension | Measures | Failure Mode |
|-----------|----------|--------------|
| Relevancy | Does answer address the question? | Off-topic responses |
| Completeness | Does answer cover all aspects? | Partial answers |
| Conciseness | Is answer free of unnecessary info? | Verbose padding |
| Coherence | Is answer logically structured? | Disjointed facts |

---

## 4. LLM Evaluation Benchmarks

### 4.1 Major Benchmarks

| Benchmark | Size | Format | What It Tests | Current SOTA |
|-----------|------|--------|---------------|-------------|
| **MMLU** | 15,908 Q | 4-choice MC | Knowledge (57 subjects) | ~88% (saturated) |
| **MMLU-Pro** | Enhanced | 10-choice MC | Harder reasoning | ~70% |
| **HumanEval** | 164 problems | Code generation | Python coding | >90% pass@1 |
| **MT-Bench** | 80 Q (multi-turn) | Open-ended | Conversation quality | GPT-4 class |
| **HELM** | 42 scenarios | Multi-metric | Holistic evaluation | Varies |
| **Chatbot Arena** | 1M+ votes | Pairwise | Human preference | Live ranking |

### 4.2 MMLU Details
- 57 subjects across STEM, Humanities, Social Sciences, Other
- Standard: 5-shot evaluation, exact-match accuracy
- **Limitation:** 6.5% error rate in questions; data contamination risk
- **Successor:** MMLU-Pro (10 choices, reasoning-focused, 16-33% harder)

### 4.3 HumanEval
```
pass@k = 1 - C(n-c, k) / C(n, k)
where n = samples, c = correct, k = attempts
```
- Standard: pass@1 (temp=0.2), pass@10, pass@100 (temp=0.8)
- Extensions: HumanEval+, MultiPL-E, EvalPlus

### 4.4 Chatbot Arena (Bradley-Terry Model)
```
P(A wins) = 1 / (1 + 10^((R_B - R_A) / 400))
```
- 1M+ votes, 90+ models
- Style-Controlled Rankings: Separate style from substance
  - Length coefficient: 0.249, Headers: 0.024, Lists: 0.031, Bold: 0.019
- Arena-Hard (automated): 500 prompts, 87.4% separability, 89.1% agreement with humans

### 4.5 HELM (7 Core Metrics)
Accuracy, Calibration, Robustness, Fairness, Bias, Toxicity, Efficiency

---

## 5. LLM-as-Judge Methodology

### 5.1 Three Evaluation Paradigms

| Paradigm | Input | Output | Best For |
|----------|-------|--------|----------|
| **Pairwise Comparison** | Two responses + question | Winner (A/B/Tie) | Model ranking |
| **Pointwise Scoring** | One response + question | Score (1-5 or 1-10) | Production monitoring |
| **Reference-Guided** | Response + question + reference | Score or winner | Factual tasks |

### 5.2 Agreement with Humans

| Judge | Agreement Rate | Source |
|-------|---------------|--------|
| GPT-4 (pairwise) | 85-87% | MT-Bench/Arena |
| Human-to-human | 81-82% | MT-Bench |
| GPT-4 (pointwise) | 80%+ exact, 95%+ within 1 point | MT-Bench |
| Prometheus-13B | 0.897 Pearson | Feedback Bench |

### 5.3 Known Biases (Quantified)

| Bias | GPT-4 | GPT-3.5 | Claude-v1 |
|------|-------|---------|-----------|
| Position consistency | 65% | 46% | 24% |
| Verbosity failure rate | 8.7% | 91.3% | 91.3% |
| Self-enhancement | +10% | N/A | +25% |

### 5.4 Bias Mitigation Strategies

| Bias | Mitigation | Effect |
|------|-----------|--------|
| Position | Swap positions + average | 65% -> 77.5% consistency |
| Verbosity | Length-Controlled AlpacaEval (GLM) | Gameability 26% -> 10% |
| Self-enhancement | PoLL (Panel of LLM evaluators) | Cohen's Kappa 0.906 vs 0.841 |

### 5.5 G-Eval Framework

**Two-Stage Process:**
1. LLM auto-generates chain-of-thought evaluation steps
2. Generated steps used in evaluation prompt; probability-weighted scoring

```
score = SUM(i=1 to n) [ p(s_i) * s_i ]
where p(s_i) = token probability of score i
```

**Performance:** Spearman 0.514 on SummEval (vs ROUGE-L: 0.165, BERTScore: 0.225)

**Implementation:** `n=20, temperature=1, top_p=1` -> sample frequencies approximate p(s_i)

### 5.6 Prometheus (Open-Source Judge)

**Prometheus 1:** Llama-2-Chat 13B
- Trained on 1K rubrics, 20K instructions, 100K responses
- Pearson correlation: 0.897 (vs GPT-4: 0.882)
- Human annotators prefer Prometheus feedback 58.62% of the time

**Prometheus 2:** Mistral-7B / Mixtral-8x7B
- Supports both direct assessment AND pairwise ranking
- Weight merging: `theta_final = 0.5 * theta_direct + 0.5 * theta_pairwise`
- Outperforms GPT-4 on Preference Bench (92.45% vs 85.50%)

### 5.7 PoLL (Panel of LLM Evaluators)

Multiple diverse smaller models collectively outperform single GPT-4 judge:
- Composition: Command R + Haiku + GPT-3.5
- Cohen's Kappa: PoLL=0.906 vs GPT-4=0.841
- **7-8x less expensive**
- Less intra-model bias through compositional diversity

### 5.8 Prompt Templates

#### MT-Bench Pairwise (pair-v2)
```
Please act as an impartial judge and evaluate the quality of the responses
provided by two AI assistants to the user question displayed below. You should
choose the assistant that follows the user's instructions and answers the user's
question better. Your evaluation should consider factors such as the helpfulness,
relevance, accuracy, depth, creativity, and level of detail of their responses.
Begin your evaluation by comparing the two responses and provide a short
explanation. Avoid any position biases and ensure that the order in which the
responses were presented does not influence your decision. Do not allow the length
of the responses to influence your evaluation. Do not favor certain names of the
assistants. Be as objective as possible. After providing your explanation, output
your final verdict by strictly following this format: "[[A]]" if assistant A is
better, "[[B]]" if assistant B is better, and "[[C]]" for a tie.
```

#### MT-Bench Single-Answer (1-10)
```
Please act as an impartial judge and evaluate the quality of the response
provided by an AI assistant to the user question displayed below. Your evaluation
should consider factors such as the helpfulness, relevance, accuracy, depth,
creativity, and level of detail of the response. Begin your evaluation by
providing a short explanation. Be as objective as possible. After providing your
explanation, you must rate the response on a scale of 1 to 10 by strictly
following this format: "[[rating]]", for example: "Rating: [[5]]".
```

#### Binary Evaluation (OpenAI Evals Pattern)
```
You are assessing a submitted answer on a given task based on a criterion.
[BEGIN DATA]
[Task]: {input}
[Submission]: {completion}
[Criterion]: {criteria}
[END DATA]
Does the submission meet the criterion? First, write out in a step by step
manner your reasoning about the criterion to be sure that your conclusion is
correct. Then print only the single character "Y" or "N" on its own line.
```

### 5.9 Implementation Best Practices

1. **Chain-of-thought before scoring** (always ask reasoning first)
2. **Binary > Likert for reliability** (more reproducible)
3. **Separate judges per criterion** (not one prompt evaluating everything)
4. **Temperature 0** for consistency
5. **Stronger model as judge** than the model being evaluated
6. **Validate against human labels** (target >90% agreement)
7. **Structured output** for reliable parsing

---

## 6. Agent Evaluation

### 6.1 AgentBench (ICLR 2024)

**8 environments, 1,091 test instances, ~13,000 interaction turns**

| Environment | Metric | GPT-4 Score |
|-------------|--------|-------------|
| Operating System | Success Rate | 42.4% |
| Database | Success Rate | 32.0% |
| Knowledge Graph | F1 Score | 58.8% |
| Digital Card Game | Win Rate | 74.5% |
| Lateral Thinking | Game Progress | 16.6% |
| House-Holding (ALFWorld) | Success Rate | 78.0% |
| Web Shopping | Reward | 61.1% |
| Web Browsing | Step Success | 29.0% |

**Key Finding:** Commercial models average 2.32+; open-source averages 0.51 (4-5x gap)

### 6.2 SWE-bench

**Task:** Given real GitHub issue + codebase, generate a patch. Evaluation: apply patch, run tests.

| Variant | Instances | Purpose |
|---------|-----------|---------|
| Full | 2,294 | Complete unfiltered set |
| Lite | 300 | Cost-reduced evaluation |
| Verified | 500 | Human-validated instances |

**Score Progression (Verified):**

| Date | System | Score |
|------|--------|-------|
| Mar 2024 | Devin | 13.86% (full) |
| Oct 2024 | Claude 3.5 Sonnet | 49.0% |
| May 2025 | Claude Opus 4 | 73.2% |
| Dec 2025 | Live-SWE-agent + Claude Opus 4.5 | 79.2% |

**Key Insight:** Scaffolding matters enormously -- same model varies 15%+ by agent framework

### 6.3 WebArena (NeurIPS 2024)

**812 tasks across 6 self-hosted web environments**

- Environments: Shopping, E-commerce CMS, Reddit, GitLab, Map, Wikipedia
- Evaluation: Functional correctness (binary pass/fail)
- Human baseline: 78.24%
- Best AI (2026): 74.3% (approaching human level)
- Original GPT-4 (2023): 14.41%

### 6.4 tau-bench (Reliability Metric)

**Novel metric: pass^k** -- success across k independent trials

```
pass^k = p^k where p = per-trial success rate
```

| Model | pass^1 | pass^8 |
|-------|--------|--------|
| GPT-4o | <50% | <25% |

**Key insight:** Models that look good on single attempts often fail on repeated trials

### 6.5 Agent Evaluation Dimensions

| Dimension | Metrics | Method |
|-----------|---------|--------|
| Task Completion | Success rate, pass@k, pass^k | Binary outcome check |
| Tool Use Accuracy | Selection, parameters, sequencing | AST matching, execution |
| Reasoning Quality | Step correctness, coherence | PRM, trajectory eval |
| Efficiency | Steps taken vs optimal | Step ratio |
| Reliability | Consistency across trials | pass^k, variance |
| Safety | Harmful actions, boundary violations | Guardrail evaluation |

---

## 7. Tool Calling Evaluation

### 7.1 Berkeley Function Calling Leaderboard (BFCL)

**Most comprehensive function calling benchmark:** ~4,441 entries across V1-V4

#### V4 Scoring Weights
| Category | Weight |
|----------|--------|
| Non-Live Single-Turn | 10% |
| Live Single-Turn | 10% |
| Irrelevance Detection | 10% |
| Multi-Turn | 30% |
| Agentic | 40% |

#### Evaluation Categories
- **Simple Function:** Single function, single call
- **Multiple Function:** Choose correct function from 2-4 options
- **Parallel Function:** Multiple simultaneous calls
- **Irrelevance Detection:** Recognize when NO tool applies
- **Multi-Turn:** State tracking across conversations

#### AST Evaluation Rules
- Function name: Exact match (dots -> underscores)
- Required params: All must be present
- Hallucinated params: Flagged as errors
- Type matching: Language-specific rules (Python int-to-float OK; Java strict)
- Strings: Case-insensitive, whitespace/punctuation normalized
- **All-or-Nothing:** Single mismatch fails entire evaluation

#### Key Findings
- Native function calling >> prompt mode (Claude Opus: 77.47% FC vs 33.47% prompt)
- Single-turn near saturation; differentiation on multi-turn/agentic
- Even 90%+ single-call accuracy = <25% reliability over 8 attempts (tau-bench)

### 7.2 ToolBench / ToolEval

**16,464 real-world REST APIs, 49 categories, 126,486 instruction instances**

| Metric | Definition |
|--------|-----------|
| **Pass Rate** | % tasks completed successfully within limited API calls |
| **Win Rate** | Pairwise preference vs ChatGPT-ReACT baseline |

Results: GPT-4 + DFSDT = 71.1% pass rate; ToolLLaMA = 66.7%

### 7.3 Common Tool Calling Failure Modes

1. Malformed function calls (syntax errors)
2. Missing URL parameters in REST requests
3. Implicit parameter conversion errors (5% vs 0.05)
4. Hallucinating non-existent tools/APIs
5. Ignoring tool outputs despite correct invocation
6. Degradation over longer tool-calling trajectories

---

## 8. Multi-Step Reasoning Evaluation

### 8.1 Process Reward Models (PRM) vs Outcome Reward Models (ORM)

| Aspect | PRM | ORM |
|--------|-----|-----|
| Evaluation granularity | Each step | Final answer only |
| Credit assignment | Dense, step-level | Sparse, end-of-trajectory |
| Error detection | Catches flawed reasoning with correct answers | Misses wrong reasoning |
| Training efficiency | 6x better sample efficiency | Simpler to implement |
| MATH performance | +5.8% over ORM | Baseline |

### 8.2 Key PRM Advances

| Model | Innovation | Result |
|-------|-----------|--------|
| **PRM800K** (OpenAI) | 800K human step labels | 78% on MATH |
| **OmegaPRM** (Google) | MCTS binary search, auto-labels | 75x efficiency, 69.4% MATH500 |
| **GenPRM** | CoT verification + code execution | 1.5B outperforms GPT-4o |
| **ThinkPRM** | Verification CoT with 1% labels | Surpasses discriminative verifiers |
| **PRIME** | Implicit process rewards from rollouts | No expensive step annotations |
| **AgentPRM** | Actor-critic for agents | 3B outperforms GPT-4o on ALFWorld |

### 8.3 Trajectory Evaluation vs Outcome Evaluation

| Dimension | Outcome Evaluation | Trajectory Evaluation |
|-----------|-------------------|----------------------|
| What it measures | Final state/result | Sequence of actions |
| Cost | Lower (one check) | Higher (every step) |
| Robustness | Tolerates creative paths | Can be brittle |
| Best for | Pass/fail gating | Safety auditing, debugging |

**Anthropic's Recommendation:** "It's often better to grade what the agent produced, not the path it took." Use outcome evaluation as primary gate; trajectory for diagnostics.

### 8.4 Progress Rate (AgentBoard)

Beyond binary success/failure:
```
Progress Rate = Subgoals achieved / Total subgoals
```
- Human-labeled intermediate checkpoints
- Reveals where agents get stuck vs completely fail
- 9 task categories, 1,013 test environments

### 8.5 Time Horizon Methodology (METR)

Measures "50%-task-completion time horizon":
- Tasks < 4 minutes: ~100% model success
- Tasks > 4 hours: <10% model success
- Doubling time: ~7 months (exponential growth)
- Current frontier: ~50 minutes of human-equivalent task time

---

## 9. Production Monitoring & Observability

### 9.1 Drift Detection

| Method | Recommendation | Key Property |
|--------|---------------|--------------|
| Domain Classifier | **Primary default** | ROC AUC; 0.5=no drift |
| Euclidean Distance | Magnitude tracking | Stable, sensitive |
| Wasserstein Distance | Numerical descriptors | Threshold 0.1 SD |
| KS Test | Small datasets (<1K) | Overly sensitive at >100K |
| PSI | Industry standard | <0.1 OK, 0.1-0.2 investigate, >=0.2 action |

**Real-World Example:** GPT-4 math accuracy dropped 84% -> 51% (March-June 2023)

#### Detection Pipeline
```
1. Embed all prompts/responses (sentence-transformers)
2. Compute daily centroid of response embeddings
3. Calculate Euclidean distance from reference centroid
4. KS test for significance
5. Alert when distance > 2 standard deviations
6. UMAP + HDBSCAN clustering for root cause
```

### 9.2 A/B Testing for LLM Systems

| Approach | Sensitivity | Cost |
|----------|------------|------|
| Between-subjects A/B | Baseline | Standard |
| Interleaving experiments | **10-100x more sensitive** | Lower |
| Bradley-Terry with style controls | SOTA for ranking | Moderate |

**Sample Size:** 1,000-16,000 votes per model for stable rankings

### 9.3 Online Evaluation Architecture

```
User Request -> LLM Response -> User
                      |
                      v (async, zero latency impact)
              Evaluation Pipeline
                      |
           +----------+----------+
           |          |          |
      LLM Judge   Embedding   Guardrails
      (sampled)   Drift        Pipeline
           |          |          |
           v          v          v
        Dashboard / Alerts / Human Review Queue
```

**Sampling Strategy:**
- High-volume: 1-10%
- Low-volume/critical: 50-100%
- Standard: 5%

### 9.4 Guardrails Pipeline (Target <200ms)

```
RegexPIIFilter (<1ms) -> BlocklistFilter (<5ms) -> ToxicityClassifier (50-100ms)
-> EmbeddingSimilarity (100-200ms) -> LLMJudge (500+ms, sampled only)
```

**Warning:** At 90% per-guard accuracy with 5 guards, compound false positive rate = ~40%. Use early-exit architecture.

### 9.5 Closed-Loop Production Pattern

```
1. Production traces logged continuously
2. Sampled traces evaluated (1-10%)
3. Failed evaluations -> human review queue
4. Validated failures added to offline test datasets
5. Offline experiments validate fixes
6. Fixes deployed back to production
```

### 9.6 Observability Platforms

| Platform | License | Best For | Stars |
|----------|---------|----------|-------|
| **Langfuse** | MIT | Annotation workflows | 29.1k |
| **Helicone** | Apache 2.0 | Minimal friction (proxy) | 5.8k |
| **Braintrust** | Proprietary | CI/CD evaluation | N/A |
| **Phoenix** | ELv2 | Embedding analysis | 12k+ |

---

## 10. Open-Source Judge Models

### 10.1 Model Comparison (RewardBench Scores)

| Model | Size | RewardBench | Key Strength |
|-------|------|-------------|-------------|
| **Skywork-Critic** | 70B | 93.3 | Highest overall score |
| **CompassJudger-2** | 7B | 90.96 | Best small model |
| **Self-Taught Evaluators** | 70B | 88.3 | Zero human annotations |
| **FLAMe-RM** | 24B | 87.8 | Lowest bias |
| **Flow-Judge** | 3.8B | Competitive | Cheapest to run |
| GPT-4o | -- | 86.7 | Baseline comparison |
| GPT-4 Turbo | -- | 85.2 | Baseline comparison |

### 10.2 When to Use What

| Scenario | Recommended Model | Rationale |
|----------|------------------|-----------|
| Custom rubric evaluation | Prometheus 2 | Designed for rubric-based scoring |
| Production at scale | PoLL (3 diverse models) | 7x cheaper, less biased |
| Reward modeling | Skywork-Critic-70B | Highest RewardBench |
| Minimal infrastructure | Flow-Judge (3.8B) | Runs on consumer GPU |
| Zero training data | Self-Taught Evaluators | Synthetic self-improvement |
| Low bias required | FLAMe-RM-24B | 0.13 avg bias vs GPT-4's 0.31 |

### 10.3 Key Trend

Open-source judges now **match or exceed GPT-4** on standard benchmarks. The gap has closed significantly in 2024-2025.

---

## 11. Cost & Latency Evaluation

### 11.1 Cost Comparison

| Approach | Cost per Evaluation | Notes |
|----------|-------------------|-------|
| GPT-4 API | $0.001-$0.01 | ~$2000 for 1000 instances |
| PoLL (3 small models) | ~$0.0035 | 7-17x cheaper than GPT-4 |
| Local Prometheus (A100) | <$30 for full benchmark | vs >$2000 GPT-4 equivalent |
| Finetuned classifiers | Millisecond latency | Upfront training investment |
| Human annotation | ~$8 per sample | Gold standard but slow |

### 11.2 Cost Optimization Strategies

| Strategy | Savings | Implementation |
|----------|---------|---------------|
| Model routing (70/30 cheap/expensive) | 63% cost reduction | Route by complexity |
| Semantic caching | 20-40% hit rate, ~$15K/month | Cache similar queries |
| PoLL (diverse model panel) | 7x cheaper | Replace single GPT-4 |
| Fine-tuned small evaluators | 100x cheaper | High-volume specific checks |
| Batch evaluation | 50% discount | Non-real-time eval |

### 11.3 Latency Benchmarks (Prometheus, 960 evaluations)

| Hardware | Tokens/sec | Total Time | Cost |
|----------|-----------|-----------|------|
| A100 80GB | 44 tok/s | 1h 20m | $2.95 |
| RTX 5000 16GB | 31.2 tok/s | 1h 46m | $1.64 |
| MacBook M3 | 20.5 tok/s | 2h 45m | Free |
| 32x AMD Epyc CPUs | 7 tok/s | 8h 05m | $9.13 |

### 11.4 Quantization Impact

| Precision | RAM Required | Pearson Correlation |
|-----------|-------------|-------------------|
| Full (FP16) | 74 GB | 0.73 (baseline) |
| Q8 (8-bit) | 16.33 GB | 0.76 |
| Q5 (5-bit) | 11.73 GB | 0.70 |

---

## 12. Google-Scale Evaluation Architecture

### 12.1 Design for 10K+ Enterprise Customers

```
                    Evaluation Control Plane
                            |
            +---------------+---------------+
            |               |               |
     Real-time Eval    Batch Eval      Human-in-Loop
     (<100ms, 5%)     (Nightly)       (Edge cases)
            |               |               |
            v               v               v
    +-------+-------+  +---+---+    +------+------+
    | Guardrails    |  | Full  |    | Expert      |
    | Pipeline      |  | RAGAS |    | Review      |
    | (PII, toxic,  |  | Suite |    | Queue       |
    |  length)      |  |       |    |             |
    +-------+-------+  +---+---+    +------+------+
            |               |               |
            v               v               v
        Spanner (Results) + BigQuery (Analytics)
                            |
                            v
                    Grafana Dashboards
                    + Alert Pipeline
```

### 12.2 Scale Requirements

| Component | Volume | Approach |
|-----------|--------|----------|
| Queries/day | 9M+ | Sample 5% for full eval = 450K evals/day |
| Eval latency budget | <100ms for real-time | Pre-computed embeddings, classifier cascade |
| Batch eval | Full corpus nightly | TPU pods, parallel RAGAS |
| Human review | 5K/day (edge cases) | Prioritized by confidence |
| Storage | Evaluation traces | BigQuery partitioned by tenant |
| Cost target | <$0.001 per query eval | Fine-tuned classifiers + sampling |

### 12.3 Multi-Tenant Evaluation

```
Per-Tenant Metrics:
- Faithfulness score distribution (p50, p95)
- Hallucination rate trend
- Retrieval precision degradation alerts
- Custom rubric scores (per vertical)

Cross-Tenant Intelligence:
- Model performance regression detection
- A/B test results aggregation
- Failure mode clustering across tenants
```

### 12.4 Evaluation Pipeline at Scale

| Stage | Method | Latency | Coverage |
|-------|--------|---------|----------|
| 1. Syntax/Format | Regex, schema validation | <1ms | 100% |
| 2. Safety | Fine-tuned classifier | 10-50ms | 100% |
| 3. Relevance | Embedding similarity | 50-100ms | 100% |
| 4. Faithfulness | NLI classifier (fine-tuned) | 50-100ms | 20% |
| 5. Full RAGAS | LLM-based evaluation | 2-5s | 5% |
| 6. Human review | Expert annotation | Hours | 0.1% |

### 12.5 Key Architecture Decisions

1. **Fine-tune small classifiers for high-volume checks** -- NLI models for faithfulness, not GPT-4
2. **Use PRMs for agent trajectories** -- 3B AgentPRM outperforms GPT-4o
3. **Bradley-Terry with style controls** for A/B testing
4. **Domain classifier for drift** (not KS test at scale)
5. **Async evaluation pipeline** -- zero latency impact on users
6. **Spanner for eval results** -- consistent reads across regions
7. **TPU inference for batch eval** -- 10x cost advantage

---

## 13. Interview Discussion Framework

### 13.1 How to Present RAG Evaluation in an Interview

**Opening Statement:**
> "RAG evaluation requires measuring both retrieval quality and generation quality independently, because failures in either component manifest differently. I use a tiered evaluation architecture with increasing cost and decreasing coverage."

**Key Points to Hit:**
1. **Separate retrieval from generation** -- diagnose which component fails
2. **Faithfulness is the critical metric** -- hallucination is the #1 production risk
3. **LLM-as-judge with bias mitigation** -- PoLL for cost, position swap for accuracy
4. **Production monitoring is different from offline eval** -- async, sampled, drift-aware
5. **Cost-quality tradeoffs are explicit** -- fine-tuned classifiers for volume, LLM for depth

### 13.2 How to Present Agent Evaluation

**Key Framework:**
> "Agent evaluation is fundamentally harder than LLM evaluation because of compounding errors, non-determinism, and the gap between trajectory correctness and outcome correctness."

**Points:**
1. **Outcome > Trajectory** for gating; trajectory for debugging
2. **pass^k reveals hidden fragility** -- 50% pass@1 = 25% pass^8
3. **Process Reward Models** outperform outcome-only by 5-9%
4. **Scaffolding matters as much as the model** (SWE-bench evidence)
5. **Time horizon is growing exponentially** -- doubling every 7 months

### 13.3 How to Present Production Monitoring

**Architecture Answer:**
> "I design a three-tier async evaluation pipeline: guardrails (100% coverage, <200ms), LLM-judge sampling (5-10%, async), and human review (edge cases). Drift detection uses domain classifiers with Euclidean distance on embedding centroids."

### 13.4 Trade-offs to Discuss

| Decision | Option A | Option B | When to Choose A |
|----------|----------|----------|-----------------|
| LLM-as-judge vs classifiers | Flexible, expensive | Fast, domain-specific | Development, low volume |
| Pairwise vs pointwise | Better discrimination | Scalable | Comparative ranking |
| Process vs outcome eval | Catches flawed reasoning | Tolerates creativity | Safety-critical |
| Full eval vs sampling | Complete coverage | Cost-effective | Low-volume, high-stakes |
| Single judge vs PoLL | Simpler | Less biased, cheaper | Prototyping |
| Reference-guided vs free | More accurate for facts | Requires gold answers | Factual domains |

### 13.5 Numbers to Quote

- GPT-4 judge agreement with humans: **85-87%** (matches inter-human)
- PoLL is **7x cheaper** with **higher Cohen's Kappa** (0.906 vs 0.841)
- PRM improves over ORM by **5.8%** on MATH, **9.1%** out-of-distribution
- SWE-bench: 1.96% (2023) -> 79.2% (2025) -- **40x improvement in 2 years**
- Drift detection: GPT-4 math dropped **84% -> 51%** in 3 months (2023)
- Evaluation cost: GPT-4 = $0.01/eval; PoLL = $0.0035; fine-tuned = $0.0001

### 13.6 Key Papers to Reference

| Paper | Key Contribution |
|-------|-----------------|
| "Judging LLM-as-a-Judge" (Zheng, 2023) | MT-Bench, bias analysis |
| "G-Eval" (Liu, 2023) | CoT + probability-weighted scoring |
| "Let's Verify Step by Step" (Lightman, 2023) | PRM >> ORM |
| "Replacing Judges with Juries" (Verga, 2024) | PoLL methodology |
| "Length-Controlled AlpacaEval" (Dubois, 2024) | Debiasing verbosity |
| "RAGAS" (Es, 2023) | RAG evaluation framework |
| Prometheus 2 (Kim, 2024) | Open-source judge matching GPT-4 |
| SWE-bench (Jimenez, 2024) | Real-world agent evaluation |
| AgentBench (Liu, 2024) | Multi-environment agent eval |
| WebArena (Zhou, 2024) | Web navigation benchmark |

---

## Summary: The Evaluation Stack

```
Layer 5: Human Expert Review (0.1% of queries)
         Gold standard, expensive, slow
         Use for: calibration, edge cases, safety

Layer 4: LLM-as-Judge (5-10% sampled)
         G-Eval, Prometheus, PoLL
         Use for: faithfulness, relevance, quality

Layer 3: Specialized Classifiers (20-50%)
         NLI for faithfulness, embedding for relevance
         Use for: targeted quality checks at speed

Layer 2: Embedding-Based Metrics (100%)
         Cosine similarity, drift detection, clustering
         Use for: distribution monitoring, anomaly detection

Layer 1: Deterministic Checks (100%)
         Format, length, PII, blocklist, schema
         Use for: guardrails, compliance, safety
```

**The key insight:** No single method is sufficient. Production systems need all five layers, with decreasing coverage but increasing depth as you go up the stack.
