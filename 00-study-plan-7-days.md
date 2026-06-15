# 7-Day Study Plan: X (Alphabet Moonshot) Staff SWE System Design

## Role Context
- **Position**: Staff Software Engineer, Early Stage Supply Chain Project
- **Company**: X (x.company) — Alphabet's moonshot factory
- **Focus**: AI-powered supply chain optimization (LLMs, RAG, Foundation Models)
- **Level**: L6/Staff equivalent

## What Makes This Interview Different

X interviews combine Google's technical rigor with moonshot-specific dimensions:
1. **Ambiguity tolerance** — problems are intentionally underspecified; they want to see you scope
2. **10x thinking** — not incremental improvements, but radical rethinking
3. **AI-native architecture** — LLM/ML systems are core, not bolted on
4. **Enterprise context** — integrating with legacy systems (SAP, Oracle ERP)
5. **Staff-level signals** — you drive architecture, mentor, handle cross-cutting concerns

## Daily Schedule

### Day 1: Foundations & Frameworks
- [ ] Read `01-framework-and-methodology.md` (the universal approach)
- [ ] Read `02-distributed-systems-fundamentals.md`
- [ ] Practice: Design a URL shortener (warm-up, apply framework)
- [ ] Practice: Design a rate limiter (capacity estimation focus)

### Day 2: Data Systems & Storage
- [ ] Read `03-data-systems-deep-dive.md`
- [ ] Practice: Design a time-series database for supply chain metrics
- [ ] Practice: Design a distributed cache with consistency guarantees
- [ ] Review trade-offs: SQL vs NoSQL vs NewSQL vs Vector DBs

### Day 3: AI/ML Systems (Critical for this role)
- [ ] Read `04-ai-ml-systems-design.md`
- [ ] Practice: Design a RAG system for enterprise document Q&A
- [ ] Practice: Design an LLM serving platform with guardrails
- [ ] Study: Embedding pipelines, vector search, retrieval strategies

### Day 4: Supply Chain + Enterprise Domain
- [ ] Read `05-supply-chain-ai-systems.md`
- [ ] Practice: Design an AI-powered demand forecasting system
- [ ] Practice: Design a real-time inventory optimization engine
- [ ] Study: ERP integration patterns, data mesh, event-driven architecture

### Day 5: Practice Problems (Full Sessions)
- [ ] Read `06-practice-problems-with-solutions.md`
- [ ] Do 2 full 45-minute mock designs (timer yourself)
- [ ] Focus on: scoping, trade-offs, Staff-level depth
- [ ] Record yourself explaining (or explain to a rubber duck)

### Day 6: Advanced Topics + Moonshot Thinking
- [ ] Read `07-advanced-topics.md`
- [ ] Read `08-moonshot-thinking-guide.md`
- [ ] Practice: Design a system that doesn't exist yet (novel problem)
- [ ] Study: How to handle "I don't know" gracefully at Staff level

### Day 7: Review & Polish
- [ ] Read `09-cheat-sheets.md` (rapid review)
- [ ] Do 1 final full mock (most challenging problem)
- [ ] Review your weak areas from the week
- [ ] Rest, sleep well, trust your preparation

## Staff-Level Evaluation Criteria (What Interviewers Look For)

| Signal | What They Want to See |
|--------|----------------------|
| **Scope** | You drive the conversation, define boundaries, make explicit trade-offs |
| **Depth** | Deep expertise in at least 2-3 areas; can go multiple levels deep on any component |
| **Breadth** | Comfortable across the full stack; understand how pieces connect |
| **Ambiguity** | Comfortable with incomplete information; make reasonable assumptions, state them |
| **Trade-offs** | Never "best" — always "best given constraints X, Y, Z" |
| **Leadership** | Proactively identify risks, propose phased rollouts, consider operational concerns |
| **Communication** | Clear, structured, collaborative; invite discussion |

## Key Differences: L5 vs L6 System Design

| Dimension | L5 (Senior) | L6 (Staff) |
|-----------|-------------|-------------|
| Scope | Single service/component | Multi-service system, cross-cutting concerns |
| Ambiguity | Clarifies requirements | Defines the problem space itself |
| Trade-offs | Identifies them | Quantifies and decides with reasoning |
| Depth | Goes deep when asked | Proactively dives where it matters |
| Operations | Mentions monitoring | Designs for observability, rollout, failure modes |
| Leadership | Executes well | Shapes direction, identifies what to NOT build |
