# Mock Interview 5: Intelligent Document Processing Platform

## Interview Format: 45-minute System Design (Staff Level)

---

## The Prompt (What the Interviewer Says)

> "Design a system that automatically processes supply chain documents — purchase orders, invoices, bills of lading, contracts, packing lists — extracts structured data, validates it against business rules, and feeds it into downstream systems. Handle any format: PDF, scanned images, emails, EDI. The system should be 99% accurate on critical fields and learn from corrections."

---

## PHASE 1: Clarifying Questions (5-8 minutes)

### Questions YOU Should Ask:

**About Volume & Types:**
1. "How many documents per day? What's the distribution across types?"
2. "Are these mostly digital-native PDFs or scanned paper (requires OCR)?"
3. "How many distinct document templates/layouts do we encounter?"
4. "Multi-language? Multilateral trade has docs in many languages."

**About Accuracy & Fields:**
5. "Which fields are 'critical' (99% accuracy required)? I'm guessing: amounts, dates, quantities, PO numbers?"
6. "What's the current process — fully manual data entry? What's the error rate today?"
7. "Is 99% accuracy per-field or per-document (all fields correct)?"
8. "How do we handle ambiguous or conflicting information within a document?"

**About Integration:**
9. "Where does extracted data go — ERP system, data warehouse, workflow engine?"
10. "Do we need to match documents to each other? (PO → Invoice → Receipt → Payment)"
11. "What validation rules exist? (e.g., invoice total = sum of line items)"
12. "Speed requirement: real-time (<1min) or batch (within hours)?"

**About Learning:**
13. "When humans correct AI mistakes, how should the system learn — immediately or in periodic retraining?"
14. "Do different customers have different document formats and rules?"
15. "How do we handle completely new document types we haven't seen before?"

### Expected Answers (Google-Scale):
- 50 MILLION documents/day across 5,000+ enterprise customers globally
- 40% digital PDF, 30% scanned paper, 15% email attachments, 10% EDI/XML, 5% photos
- 500K+ distinct templates/layouts (long tail: new formats discovered daily)
- Critical fields: amounts (99.5%), dates (99.5%), PO/invoice numbers (99.9%), line items (98%)
- Currently: replacing a fragmented market of manual + legacy OCR solutions
- Multi-tenant: each customer has unique formats, rules, validation logic, and ERP schemas
- Feeds into: ERP, payment systems, compliance engines, audit trails, analytics platforms
- Processing time: <60 seconds for 95% of documents (real-time for business workflows)
- Learn from corrections within 1 hour (continuous online learning at scale)
- Multi-language: 40+ languages, including CJK, Arabic (RTL), mixed-language documents

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me size this at Google scale:

THROUGHPUT:
- 50M docs/day = ~580 docs/sec sustained
- But bursty: business hours across global timezones = ~800 docs/sec base
- Peak burst: 5x = 4,000 docs/sec (quarter-end, audit seasons, tax deadlines)
- Absolute max design target: 10,000 docs/sec (headroom + growth)

PROCESSING BUDGET:
- <60 seconds SLO for 95% of documents (p95)
- <5 seconds for known templates with high-confidence extraction (60% of docs)
- <30 seconds for novel layouts requiring VLM analysis (30% of docs)
- <120 seconds for complex multi-page documents with tables (10% of docs)
- Bottleneck: GPU inference for VLM/OCR, NOT I/O or network

ACCURACY MATH:
- 99.5% accuracy on amounts = 1 error per 200 documents
- At 50M docs/day = 250K documents flagged for review
- BUT: auto-approval rate target is 92% → only 4M docs need human review
- 4M reviews/day at avg 30 sec per review = 33,000 reviewer-hours/day
- Crowd-sourced + in-house reviewers across timezones = feasible with 5,000 reviewers
- Each reviewer: 100-150 docs/hour = 800-1200 docs/day
- KEY INSIGHT: Every 1% improvement in auto-approval = 500K fewer reviews = 40 fewer FTEs

STORAGE:
- Avg document: 250KB (PDF/image) + 8KB (extracted JSON) + 2KB (metadata/audit)
- Daily originals: 50M × 250KB = 12.5 TB/day
- Daily extracted: 50M × 10KB = 500 GB/day
- Keep originals 10 years (compliance): 12.5 TB × 365 × 10 = 45 PB
- Extracted data (queryable): 500 GB × 365 × 10 = 1.8 PB
- Training data (corrections + ground truth): ~500 TB accumulated
- TOTAL STORAGE: ~50 PB (lifecycle-managed with cold/archive tiers)

COST:
- OCR/extraction per doc: $0.01-0.05 (depending on model)
- 200K × $0.03 = $6K/day = $180K/month
- Compare to: 40 manual data entry staff × $5K/month = $200K/month
- ROI is clear even before accuracy improvements

Good framing?"
```

---

## PHASE 3: High-Level Architecture (10 minutes)

### Architecture Diagram:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DOCUMENT INGESTION                                  │
│                                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────┐│
│  │  Email   │  │  SFTP    │  │  API     │  │  Scanner │  │  EDI  ││
│  │ (mailbox)│  │  (watch) │  │ (upload) │  │  (agent) │  │(parse)││
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬───┘│
│       └──────────────┴──────────────┴──────────────┴────────────┘    │
│                                     │                                 │
│                                     ▼                                 │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Document Queue (SQS/Pub-Sub)                                    │  │
│  │ - Priority: urgent POs > standard invoices > archival           │  │
│  │ - Retry: 3x with exponential backoff                            │  │
│  │ - DLQ: failed after retries → human review                     │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    PROCESSING PIPELINE                                 │
│                                                                        │
│  STAGE 1: PREPROCESSING                                               │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ - File type detection (PDF vs image vs email)                    │  │
│  │ - PDF rendering (convert to images for OCR if needed)            │  │
│  │ - Image enhancement (deskew, denoise, contrast for scanned docs) │  │
│  │ - Page splitting (multi-page docs → individual pages)            │  │
│  │ - Language detection                                              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  STAGE 2: DOCUMENT UNDERSTANDING (Multi-Model)                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │ MODEL A: Layout-Aware (LayoutLMv3 / DocFormer)           │    │  │
│  │  │ - Understands document structure (tables, headers, etc)  │    │  │
│  │  │ - Fine-tuned on supply chain document corpus              │    │  │
│  │  │ - Best for: structured forms, tables, standard layouts    │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │ MODEL B: Vision-Language Model (GPT-4V / Claude Vision)  │    │  │
│  │  │ - Understands arbitrary document formats                  │    │  │
│  │  │ - Zero-shot capability for new templates                  │    │  │
│  │  │ - Best for: irregular layouts, handwritten annotations    │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐    │  │
│  │  │ MODEL C: Template Matching (trained per document type)   │    │  │
│  │  │ - Highest accuracy for known templates                    │    │  │
│  │  │ - Fast, cheap inference                                   │    │  │
│  │  │ - Requires labeled examples per template                  │    │  │
│  │  └─────────────────────────────────────────────────────────┘    │  │
│  │                                                                  │  │
│  │  ROUTING LOGIC:                                                  │  │
│  │  - Known template (>80% confidence) → Model C (fast, accurate)  │  │
│  │  - Known doc type, unknown template → Model A (layout-aware)    │  │
│  │  - Unknown everything → Model B (VLM, zero-shot)                │  │
│  │  - Low confidence from any → escalate to human                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  STAGE 3: EXTRACTION & STRUCTURING                                    │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Output schema (enforced via structured output / function calling):│  │
│  │ {                                                                 │  │
│  │   "document_type": "invoice",                                     │  │
│  │   "header": {                                                     │  │
│  │     "invoice_number": {"value": "INV-2026-5678", "confidence": 0.99},│
│  │     "date": {"value": "2026-06-10", "confidence": 0.98},         │  │
│  │     "total_amount": {"value": 45230.00, "confidence": 0.97},     │  │
│  │     "currency": {"value": "USD", "confidence": 0.99},            │  │
│  │     "vendor": {"value": "Acme Supply Co", "confidence": 0.95}    │  │
│  │   },                                                              │  │
│  │   "line_items": [                                                 │  │
│  │     {"sku": "SKU-789", "qty": 100, "unit_price": 45.23,         │  │
│  │      "line_total": 4523.00, "confidence": 0.96}                  │  │
│  │   ],                                                              │  │
│  │   "metadata": {                                                   │  │
│  │     "page_count": 2,                                              │  │
│  │     "model_used": "template_match_v3",                            │  │
│  │     "processing_time_ms": 2300                                    │  │
│  │   }                                                               │  │
│  │ }                                                                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  STAGE 4: VALIDATION & ENRICHMENT                                     │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Rule-based validation:                                           │  │
│  │ - Arithmetic: line_total = qty × unit_price (within $0.01)       │  │
│  │ - Sum: total_amount = Σ(line_totals) + tax + shipping            │  │
│  │ - Referential: PO number exists in our system                    │  │
│  │ - Format: dates are valid, amounts are positive                   │  │
│  │ - Business: vendor is in approved supplier list                   │  │
│  │                                                                   │  │
│  │ Cross-reference enrichment:                                       │  │
│  │ - Match vendor name to entity in master data (fuzzy matching)     │  │
│  │ - Match SKU descriptions to our product catalog                   │  │
│  │ - Look up PO to verify quantities and prices match                │  │
│  │                                                                   │  │
│  │ Confidence routing:                                                │  │
│  │ - ALL fields high confidence + validation passes → AUTO-APPROVE   │  │
│  │ - ANY critical field low confidence → HUMAN REVIEW                │  │
│  │ - Validation fails → FLAG + HUMAN REVIEW                          │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    HUMAN-IN-THE-LOOP LAYER                             │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Review Queue UI:                                                 │  │
│  │ - Show original document side-by-side with extracted fields      │  │
│  │ - Highlight low-confidence fields (red/yellow)                   │  │
│  │ - Allow inline correction (click field, type correct value)      │  │
│  │ - Keyboard shortcuts for speed (Tab, Enter to confirm)           │  │
│  │ - Smart ordering: highest business impact first                  │  │
│  │                                                                   │  │
│  │ Review effort metrics:                                            │  │
│  │ - Avg time per document review: 45 seconds                       │  │
│  │ - Auto-approval rate: target 95%                                  │  │
│  │ - Fields corrected per reviewed doc: avg 1.2                      │  │
│  └────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬──────────────────────────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    LEARNING & IMPROVEMENT LAYER                        │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Active Learning Pipeline:                                         │  │
│  │                                                                   │  │
│  │ Human corrections → Correction Store → Nightly Retraining        │  │
│  │                         │                                         │  │
│  │                         ▼                                         │  │
│  │     Template Recognition: "Is this a new template?"              │  │
│  │       YES → Create template, fine-tune Model C                   │  │
│  │       NO → Add to existing template's training data              │  │
│  │                                                                   │  │
│  │ Feedback loop metrics:                                            │  │
│  │ - Track accuracy per template over time (should improve)         │  │
│  │ - Track correction patterns (systematic errors → model gap)      │  │
│  │ - New template detection (clustering unseen layouts)              │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## PHASE 4: Deep Dives (20-25 minutes)

### Deep Dive 1: Multi-Model Strategy & Routing

```
"Why three models instead of just using a VLM for everything?

COST-ACCURACY-SPEED MATRIX:
┌─────────────────┬──────────┬──────────┬──────────┬──────────────┐
│ Model           │ Accuracy │ Speed    │ Cost/doc │ When to Use   │
├─────────────────┼──────────┼──────────┼──────────┼──────────────┤
│ Template Match  │ 99.5%    │ 200ms    │ $0.001   │ Known layouts │
│ (fine-tuned)    │          │          │          │ (80% of docs) │
├─────────────────┼──────────┼──────────┼──────────┼──────────────┤
│ LayoutLM        │ 97%      │ 1-2s     │ $0.01    │ Known types,  │
│ (layout-aware)  │          │          │          │ new layouts   │
├─────────────────┼──────────┼──────────┼──────────┼──────────────┤
│ VLM (Claude/GPT)│ 95%      │ 5-15s    │ $0.03-   │ Unknown docs, │
│ (zero-shot)     │          │          │  $0.10   │ complex cases │
└─────────────────┴──────────┴──────────┴──────────┴──────────────┘

ROUTING DECISION TREE:
┌─────────────────────────────────────────────────────────────────┐
│  Document arrives                                                │
│       │                                                          │
│       ▼                                                          │
│  Template Fingerprint (hash of layout structure)                  │
│       │                                                          │
│       ├─ MATCH (confidence > 0.9) → Template Model (fast, cheap) │
│       │                                                          │
│       ├─ PARTIAL MATCH (0.6-0.9) → LayoutLM (moderate)           │
│       │                                                          │
│       └─ NO MATCH → VLM (expensive, but handles anything)        │
│                                                                  │
│  After extraction, regardless of model:                          │
│  - Validation rules run (arithmetic, referential)                │
│  - If validation fails + confidence was high → possible          │
│    model degradation → alert ML team                             │
└─────────────────────────────────────────────────────────────────┘

TEMPLATE FINGERPRINTING (how we quickly identify known templates):
- Extract: page dimensions, text block positions, logo hash, header pattern
- Create structural fingerprint (ignoring content, capturing layout)
- Compare to library of known fingerprints (cosine similarity)
- 200K docs/day × 200ms fingerprinting = 11 hours (easily parallelizable)

This tiered approach saves 80% of cost vs using VLM for everything,
while maintaining accuracy by routing hard cases to capable models.
"
```

---

### Deep Dive 2: Line Item Extraction (The Hardest Part)

```
"Line items in tables are the most challenging extraction task. Here's why and how:

CHALLENGES:
1. Table layouts vary wildly (column order, merged cells, multi-row items)
2. Some PDFs have invisible table structure (text-based, not actual tables)
3. Scanned docs: OCR can merge/split table cells incorrectly
4. Multi-page tables (continuation across pages)
5. Sub-line items (variants, specifications nested under main item)

APPROACH: TABLE DETECTION → STRUCTURE RECOGNITION → CELL EXTRACTION

┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Table Detection                                          │
│ - Object detection model (fine-tuned DETR) identifies table regions│
│ - Handles: bordered tables, borderless tables, form-style layouts │
│ - Output: bounding boxes of table regions on each page            │
│                                                                   │
│ STEP 2: Structure Recognition                                     │
│ - Identify rows, columns, headers                                 │
│ - Handle multi-line rows (description wrapping)                   │
│ - Handle spanning cells (merged)                                  │
│ - Output: grid structure with cell bounding boxes                 │
│                                                                   │
│ STEP 3: Cell Content Extraction                                   │
│ - OCR each cell (or extract text if digital PDF)                  │
│ - Classify each column: SKU | description | quantity | unit_price │
│   | line_total | UOM | tax | discount                             │
│ - Normalize: "1,234.56" → 1234.56, "12 EA" → {qty: 12, uom: EA} │
│                                                                   │
│ STEP 4: Semantic Matching                                         │
│ - Match extracted SKU to our product catalog (fuzzy matching)     │
│ - Match description to known products (embedding similarity)      │
│ - Resolve ambiguity: "Milk 1L" → which of 15 milk SKUs?          │
│   → Use context: vendor, price, historical orders                  │
└─────────────────────────────────────────────────────────────────┘

ARITHMETIC VALIDATION (catches ~40% of extraction errors):
- For each line: qty × unit_price should = line_total (±$0.01)
- Sum of line_totals + tax + shipping should = invoice total
- If math doesn't check out → flag specific cells for review
- This is a FREE accuracy boost — no ML needed, just arithmetic

MULTI-PAGE TABLE HANDLING:
- Detect: header repeats at top of page 2 → continuation
- Merge: concatenate rows, re-validate structure
- Challenge: last row of page 1 + first row of page 2 might be same item
- Solution: check if row is "complete" (has qty, price, total) before closing
"
```

---

### Deep Dive 3: Learning from Corrections (Active Learning)

```
"The system should get better over time. Here's the learning architecture:

┌─────────────────────────────────────────────────────────────────┐
│            ACTIVE LEARNING LOOP                                   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ CORRECTION CAPTURE                                        │    │
│  │                                                           │    │
│  │ When human corrects a field:                              │    │
│  │ Store: {                                                  │    │
│  │   document_id, page_image, field_name,                    │    │
│  │   predicted_value, correct_value,                         │    │
│  │   model_used, confidence_was, template_id                 │    │
│  │ }                                                         │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ ANALYSIS (nightly batch)                                  │    │
│  │                                                           │    │
│  │ 1. Error clustering:                                      │    │
│  │    - Group corrections by pattern (same field, same       │    │
│  │      template, same error type)                           │    │
│  │    - "Template X always gets vendor name wrong"           │    │
│  │    - "Date format DD/MM/YYYY confused with MM/DD/YYYY"    │    │
│  │                                                           │    │
│  │ 2. Template discovery:                                    │    │
│  │    - Cluster documents that went to VLM (expensive path)  │    │
│  │    - If cluster > 10 docs with similar layout → new template│   │
│  │    - Auto-create template, fine-tune extraction model     │    │
│  │                                                           │    │
│  │ 3. Confidence calibration:                                │    │
│  │    - Is confidence score well-calibrated?                 │    │
│  │    - "99% confidence" should mean 99% are actually correct│    │
│  │    - Adjust thresholds based on observed accuracy         │    │
│  └──────────────────────────┬──────────────────────────────┘    │
│                             │                                     │
│                             ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ RETRAINING (weekly, automated)                            │    │
│  │                                                           │    │
│  │ 1. Add corrected samples to training set                  │    │
│  │ 2. Fine-tune template models on expanded data             │    │
│  │ 3. Evaluate on held-out test set (MUST improve, not regress)│  │
│  │ 4. Shadow deployment: run new model alongside old         │    │
│  │ 5. If new model better on all metrics → promote           │    │
│  │ 6. If regression on any template → investigate before deploy│  │
│  │                                                           │    │
│  │ GUARD RAILS:                                              │    │
│  │ - Never auto-deploy model worse than current on ANY metric│    │
│  │ - Minimum training samples per template: 50               │    │
│  │ - Human ML engineer reviews weekly retrain results        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  EXPECTED IMPROVEMENT CURVE:                                      │
│  - Week 1: 85% auto-approval rate (cold start)                   │
│  - Month 1: 90% (learned common templates)                       │
│  - Month 3: 94% (long tail of templates captured)                │
│  - Month 6: 96% (refinements, edge cases)                        │
│  - Steady state: 95-97% (new vendors/formats always arriving)    │
└─────────────────────────────────────────────────────────────────┘
"
```

---

## PHASE 5: Wrap-Up (5 minutes)

### Three-Way Document Matching
```
"A critical downstream use case: matching PO → Invoice → Receipt

Why: Detect discrepancies (overcharging, short shipments, unauthorized purchases)

PO says: 100 units of SKU-789 at $45/unit = $4,500
Invoice says: 100 units of SKU-789 at $47/unit = $4,700  ← PRICE MISMATCH
Receipt says: 95 units of SKU-789  ← QUANTITY MISMATCH

The document processing system extracts structured data that enables 
this matching. The matching engine is a separate service that:
1. Correlates documents by PO number
2. Compares field values across documents
3. Flags discrepancies above threshold
4. Routes to AP team for resolution

This is where the real business value lives — not just in extraction,
but in the automated reconciliation it enables."
```

### Key Trade-offs:
```
"1. ACCURACY vs THROUGHPUT:
   - Could achieve 99.9% with 3 models + human review on everything
   - But that defeats the purpose (saving human time)
   - My choice: tiered confidence → auto-approve high-confidence, 
     human-review low-confidence only

2. GENERALIZATION vs SPECIALIZATION:
   - VLM generalizes to any document (expensive, slower)
   - Template model specializes per format (cheap, fast, limited)
   - My choice: specialize where possible, generalize where necessary

3. LEARNING SPEED vs STABILITY:
   - Could retrain daily for fastest learning
   - But risks instability (bad batch of corrections → model regression)
   - My choice: weekly retrain with shadow evaluation before deployment"
```

---

## Scoring Rubric

| Criterion | Strong Signal | Did I Hit It? |
|-----------|--------------|:-------------:|
| Pipeline design | Clear stages: preprocess → classify → extract → validate | ☐ |
| Multi-model strategy | Tiered by cost/accuracy, intelligent routing | ☐ |
| Table extraction | Specific approach for the hardest part (line items) | ☐ |
| Validation | Arithmetic checks, referential integrity, business rules | ☐ |
| Human-in-the-loop | Smart routing, efficient UI, minimal reviewer burden | ☐ |
| Active learning | Correction capture, clustering, automated retraining | ☐ |
| Business value | 3-way matching, auto-approval rate, cost savings | ☐ |
| Staff signals | Phased rollout, metrics-driven, operational concerns | ☐ |
# ENHANCED: Document Processing — Trade-offs, Alternatives & Scale

## Addendum to Mock Interview 5 — Read alongside the main file

---

## DETAILED SCALE ESTIMATES (GOOGLE-SCALE)

### Processing Pipeline Throughput
```
DOCUMENTS: 50M/day target (across 5,000 enterprise customers)

PROCESSING TIME BUDGET (<60 sec SLO for p95):
  Tier 1 — Known template, high confidence (60% = 30M docs/day):
  - Template match: 100ms
  - Field extraction (regex + lightweight model): 500ms
  - Validation: 200ms
  - Total: <1 second per doc
  
  Tier 2 — Known layout family, needs ML extraction (30% = 15M docs/day):
  - OCR (if scanned): 2 seconds
  - LayoutLM/DocFormer inference: 3 seconds
  - Field extraction + validation: 1 second
  - Total: 3-6 seconds per doc
  
  Tier 3 — Novel/complex documents requiring VLM (10% = 5M docs/day):
  - OCR + layout analysis: 3 seconds
  - VLM inference (Gemini Pro Vision): 5-15 seconds
  - Multi-page correlation: 5 seconds
  - Total: 15-25 seconds per doc

THROUGHPUT DESIGN:
  50M docs/day ÷ 20 active hours (global) = 2.5M docs/hour = 694 docs/sec avg
  Peak (quarter-end/tax season): 5x = 3,472 docs/sec
  
  Tier 1 processing (30M/day, <1s each):
  - 30M ÷ 72,000s (20 hours) = 417 docs/sec avg
  - Need: 417 concurrent processing slots (CPU-only, cheap)
  - 50 GKE pods with 8 vCPU each handles this at 80% utilization
  
  Tier 2 processing (15M/day, ~5s each):
  - 15M ÷ 72,000s = 208 docs/sec avg
  - Concurrent: 208 × 5s = 1,040 processing slots needed
  - Mix of CPU (OCR) + GPU (LayoutLM): 100 GPU pods (4× A100 each)
  - Batch inference: groups of 32 docs per GPU = efficient utilization
  
  Tier 3 processing (5M/day, ~15s each):
  - 5M ÷ 72,000s = 69 docs/sec avg
  - Concurrent: 69 × 15s = 1,035 VLM inference slots
  - With batching and 4× throughput per GPU: 260 GPU pods
  - OR: Use Vertex AI Gemini API (serverless, pay-per-call)
    5M × $0.03/doc = $150K/day via API vs $200K/month self-hosted
    DECISION: Self-host for cost + latency control at this scale

COST:
┌────────────────────────────────────────────────────────────────────────────┐
│ Component              │ Cost/doc  │ Daily (50M)  │ Monthly Cost            │
├────────────────────────┼───────────┼──────────────┼─────────────────────────┤
│ Tier 1 (template)      │ $0.0005   │ $15K         │ $450K                   │
│ Tier 2 (LayoutLM)      │ $0.005    │ $75K         │ $2.25M                  │
│ Tier 3 (VLM/Gemini)    │ $0.02     │ $100K        │ $3.0M                   │
│ OCR (Google Doc AI)    │ $0.003    │ $67K (45%)   │ $2.0M                   │
│ Storage (GCS + BQ)     │ $0.0002   │ $10K         │ $300K                   │
│ Human review (4M/day)  │ $0.30     │ $1.2M        │ $36M (biggest cost!)    │
│ Compute (GKE + GPU)    │ $0.003    │ $150K        │ $4.5M                   │
├────────────────────────┼───────────┼──────────────┼─────────────────────────┤
│ TOTAL                  │ ~$0.033   │ ~$1.6M/day   │ ~$48M/month             │
│ vs manual entry only   │ $3-5/doc  │ $150-250M/day│ Would be $4.5-7.5B/mo!  │
│ SAVINGS                │ ~99%      │              │                         │
└────────────────────────────────────────────────────────────────────────────┘

CRITICAL INSIGHT: Human review is 75% of total cost!
- Every 1% improvement in auto-approval = $360K/month savings
- This is WHY active learning and continuous model improvement is the #1 priority
- At 99% auto-approval (target Year 3): human cost drops to $3.6M/month
  Total system cost: ~$15M/month → $180M/year
  Revenue (5000 customers × avg $100K/year): $500M/year → 64% gross margin

BREAK-EVEN vs ALTERNATIVES:
  Our cost: $0.033/doc (at 92% auto-approval)
  Manual data entry (offshore): $3-5/doc
  Legacy OCR + manual review: $0.50-1.00/doc
  Savings vs legacy: 15-30x cheaper
  Savings vs manual: 100-150x cheaper
  Customer ROI: Processes 50M docs that would cost $150M+ manually for $48M total
```

### Accuracy Deep Dive
```
WHAT "99% ACCURACY" ACTUALLY MEANS:

Per-field accuracy:
  - Amount field correct: 99% → 1 error per 100 docs
  - ALL 10 fields correct on one doc: 0.99^10 = 90.4% per-doc accuracy
  - This means 10% of docs have at least one wrong field!

BETTER FRAMING:
  - 99% on critical fields (amount, date, PO number): 3 fields
  - 95% on non-critical fields (description, address, notes): 7 fields  
  - P(all critical correct) = 0.99³ = 97%
  - P(auto-approvable) = P(all critical + all non-critical) = 0.97 × 0.95⁷ = 68%
  
  WAIT — 68% auto-approval is too low! 

FIX: Confidence-based routing is the solution:
  - Extract ALL fields
  - Per-field confidence scoring
  - Auto-approve if: ALL critical fields > 0.98 AND non-critical > 0.90
  - In practice: ~95% of documents pass this bar for KNOWN templates
  - Unknown templates: ~70% pass → rest go to human review
  
  WEIGHTED AUTO-APPROVAL:
  - Known templates (80% of docs): 95% auto-approved
  - Unknown templates (20% of docs): 70% auto-approved
  - Overall: 0.8×0.95 + 0.2×0.70 = 90% auto-approved

  HUMAN REVIEW VOLUME:
  - 200K docs × 10% needing review = 20K docs/day
  - Avg review time: 45 seconds (field corrections, not full data entry)
  - Reviewer capacity: 80 docs/hour × 8 hours = 640 docs/reviewer/day
  - Team needed: 20K ÷ 640 = 32 reviewers
  - Compare to previous 80 full-time data entry staff → 60% reduction
```

---

## ALTERNATIVE EXTRACTION APPROACHES

### Approach A: Template-Based (Traditional OCR + Rules)
```
DESIGN:
  For each known document template:
  - Define field locations (bounding boxes)
  - Define extraction rules (regex for amounts, date parsing)
  - Template matching via fingerprint

HOW IT WORKS:
  1. Fingerprint document layout → match to template library
  2. Apply pre-defined extraction zones (x1,y1,x2,y2 for each field)
  3. OCR within each zone
  4. Apply rules: regex for PO numbers, amount parsing, date parsing

PROS:
+ Extremely fast (<200ms per doc once template matched)
+ Cheapest per doc ($0.001)
+ 99.5%+ accuracy for known templates (zones are precise)
+ Fully deterministic (same doc → same output every time)
+ No GPU needed

CONS:
- EVERY new template needs manual zone definition (hours of setup)
- 500 templates × 10-15 fields × manual config = 5,000-7,500 field configs
- Template variations break it (vendor updates their invoice layout)
- Cannot handle unknown templates (fails silently or hard)
- Doesn't understand content (can't resolve ambiguity)
- Maintenance burden grows linearly with template count

WHEN TO CHOOSE: 
  High-volume single-source documents (e.g., process ALL invoices from Walmart)
  Legacy systems that already have template configs
  
SCALE LIMIT: ~200-500 templates before maintenance becomes unsustainable
```

### Approach B: Layout-Aware AI (LayoutLM, DocFormer, Donut)
```
DESIGN:
  Pre-trained model that understands document layout + text jointly
  Fine-tuned on supply chain documents for field extraction
  Input: document image + OCR text + bounding boxes
  Output: field values with positions

HOW IT WORKS:
  1. OCR full document → text + position for every word
  2. Feed (text, position, image) to LayoutLMv3 model
  3. Model classifies each word/region as field type
  4. Aggregate classified words into field values

PROS:
+ Handles layout variations (same fields in different positions)
+ Generalizes across templates within same doc type
+ Single model handles hundreds of templates
+ Can extract from new templates with zero-shot (lower accuracy)
+ Moderate cost and latency (~2 seconds per doc)

CONS:
- Requires training data (100+ labeled examples per doc type)
- Accuracy lower than template-based for known templates (97% vs 99.5%)
- Can be confused by unusual layouts or overlapping fields
- Model updates require retraining pipeline
- Less accurate on handwritten or low-quality scans

WHEN TO CHOOSE:
  Main workhorse for known document types (invoices, POs, BOLs)
  When you have labeled training data (or can create it from corrections)
```

### Approach C: Vision-Language Model (Claude, GPT-4V) — Zero-Shot
```
DESIGN:
  Feed document image directly to VLM with structured output prompt
  Prompt defines desired output schema
  Model extracts fields from visual understanding

HOW IT WORKS:
  1. Render document to image(s)
  2. Prompt: "Extract the following fields from this invoice: [schema]
     Output as JSON. Include confidence for each field."
  3. VLM returns structured extraction

PROS:
+ Zero-shot (no training data needed for new document types!)
+ Handles ANY layout (never seen before = still works)
+ Understands context (can resolve ambiguous fields using reasoning)
+ Multi-language out of the box
+ Handles complex documents (multi-page, mixed content)

CONS:
- Most expensive ($0.03-0.10 per doc)
- Slowest (5-15 seconds per doc)
- Non-deterministic (same doc might get slightly different output)
- Can hallucinate field values (especially for poor quality scans)
- Rate-limited by API provider
- Accuracy depends on prompt engineering

WHEN TO CHOOSE:
  - Unknown/new document templates (zero-shot capability)
  - Low-volume, high-value documents (contracts, custom forms)
  - Prototyping before investing in LayoutLM fine-tuning
  - Fallback when specialized models are uncertain
```

### Approach D: Hybrid Tiered — MY CHOICE
```
TIER ROUTING LOGIC:

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Document arrives                                                │
│       │                                                          │
│       ▼                                                          │
│  Template fingerprint (layout hash, logo detection)              │
│       │                                                          │
│       ├─ STRONG MATCH (>95% confidence, >100 historical examples)│
│       │   → TIER 1: Template-based extraction                    │
│       │   Cost: $0.001, Time: 200ms, Accuracy: 99.5%            │
│       │   Volume: ~60% of docs                                   │
│       │                                                          │
│       ├─ PARTIAL MATCH (60-95%, known doc TYPE but new layout)   │
│       │   → TIER 2: LayoutLM extraction                          │
│       │   Cost: $0.005, Time: 2s, Accuracy: 97%                  │
│       │   Volume: ~25% of docs                                   │
│       │                                                          │
│       └─ NO MATCH (<60%, completely new)                         │
│           → TIER 3: VLM extraction (Claude/GPT-4V)               │
│           Cost: $0.05, Time: 10s, Accuracy: 92%                  │
│           Volume: ~15% of docs                                   │
│                                                                   │
│  POST-EXTRACTION (all tiers):                                    │
│  → Validation rules (arithmetic, format, referential)            │
│  → Confidence assessment                                         │
│  → Auto-approve (>95% confidence) or Human review                │
│                                                                   │
│  MIGRATION PATH:                                                 │
│  As VLM-processed docs get corrections:                          │
│  → Accumulate training data for that template                    │
│  → At 50+ examples: train LayoutLM (moves from Tier 3 → Tier 2) │
│  → At 200+ examples: create template config (Tier 2 → Tier 1)   │
│  → Net effect: Tier 3 shrinks over time, cost decreases          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

COST OPTIMIZATION OVER TIME:
  Month 1: 40% T1, 30% T2, 30% T3 → avg $0.020/doc
  Month 6: 60% T1, 25% T2, 15% T3 → avg $0.011/doc  
  Month 12: 75% T1, 17% T2, 8% T3 → avg $0.007/doc
  
  The system GETS CHEAPER as it learns more templates!
```

---

## TRADE-OFF MATRICES

### OCR Engine Selection
| Engine | Accuracy | Speed | Cost | Languages | Handwriting | Integration |
|--------|----------|-------|------|-----------|-------------|-------------|
| Google Document AI | 99%+ | Fast | $1.50/1K pages | 200+ | Good | GCP native |
| AWS Textract | 98% | Fast | $1.50/1K pages | 5 | Medium | AWS native |
| Azure Form Recognizer | 99% | Fast | $1/1K pages | 100+ | Good | Azure native |
| Tesseract (open-source) | 92% | Slow | Free | 100+ | Poor | Self-hosted |
| PaddleOCR (open-source) | 96% | Medium | Free | 80+ | Medium | Self-hosted |

**MY CHOICE: Google Document AI** (GCP environment, highest accuracy, table detection built-in)
**Fallback: PaddleOCR** for cost optimization on simple, clean documents

### Confidence Threshold Trade-offs
```
HIGH THRESHOLD (0.98): Conservative
  - Auto-approve rate: 75%
  - Error rate in auto-approved: 0.1%
  - Human review volume: HIGH (50K docs/day)
  - Best for: Regulated industries, financial documents

MEDIUM THRESHOLD (0.93): Balanced — MY CHOICE
  - Auto-approve rate: 90%
  - Error rate in auto-approved: 0.5%
  - Human review volume: MODERATE (20K docs/day)
  - Best for: General enterprise (cost-accuracy balance)

LOW THRESHOLD (0.85): Aggressive
  - Auto-approve rate: 97%
  - Error rate in auto-approved: 2%
  - Human review volume: LOW (6K docs/day)
  - Best for: Low-stakes documents, high-volume processing

DYNAMIC THRESHOLD:
  Per-customer: Some customers tolerate more errors (fast-moving retail)
  Per-field: Amount thresholds higher than description thresholds
  Per-template: Known accurate templates can have lower thresholds
```

### Active Learning Strategy Trade-offs
```
OPTION A: IMMEDIATE LEARNING
  Correction → immediately update extraction rules
  Pro: Fastest adaptation
  Con: One bad correction can break subsequent documents
  Risk: HIGH (single point of failure in corrections)

OPTION B: BATCH LEARNING (weekly retrain)
  Corrections accumulate → weekly model fine-tune → A/B test → deploy
  Pro: Stable, validated before deployment, catches bad corrections
  Con: 7-day lag before improvements take effect
  Risk: LOW (validated)

OPTION C: CONTINUOUS LEARNING (streaming)
  Corrections → update model with online learning (every hour)
  Pro: Fast adaptation (hours not days), continuous improvement
  Con: Model can drift, harder to reproduce/debug
  Risk: MEDIUM (needs monitoring for degradation)

MY CHOICE: OPTION B (weekly retrain) with OPTION A for template configs only
  - Template zone configs: update immediately (deterministic, low risk)
  - LayoutLM fine-tuning: weekly batch (needs GPU, validation required)
  - VLM prompts: update monthly (prompt engineering requires testing)
  
  WHY NOT CONTINUOUS:
  - Supply chain documents have regulatory implications
  - An unstable model that gets worse is worse than a stable slightly stale model
  - Weekly cycle gives time for human ML engineer to review training data quality
  - A/B testing ensures no regression (critical for enterprise trust)
```

---

## MULTI-TENANT CONSIDERATIONS

### Document Isolation
```
PROBLEM: Customer A's invoice templates ≠ Customer B's
         But underlying ML models should benefit from cross-customer learning

DESIGN:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│ SHARED (cross-customer):                                         │
│ - Base LayoutLM model (pre-trained on public datasets)           │
│ - OCR engine (stateless, no customer data retained)              │
│ - General document type classifier                               │
│ - Validation rule templates (arithmetic, date format)            │
│                                                                   │
│ PER-CUSTOMER (isolated):                                         │
│ - Template fingerprint library (their specific vendor layouts)   │
│ - Fine-tuned extraction model (optional: customer-specific)      │
│ - Field mapping config (their PO format → canonical format)      │
│ - Validation rules (their business rules: max order $X)          │
│ - Correction history (training data for their documents)         │
│ - Extracted data (obviously)                                     │
│                                                                   │
│ HYBRID (shared model, isolated input):                           │
│ - Cross-customer model trained on anonymized structural features │
│   (layout, positions — NOT content)                              │
│ - Per-customer "adapter" that handles their specific field names  │
│ - This way: structural learning transfers, content stays private  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

TRADE-OFF: Privacy vs Accuracy
  - Fully isolated (per-customer models): Highest privacy, slower learning
  - Fully shared: Fastest learning, privacy risk
  - MY CHOICE: Shared structure, isolated content (best of both)
```

---

## INTEGRATION WITH DOWNSTREAM SYSTEMS

### Three-Way Match Architecture
```
The REAL business value: automated reconciliation

PO (what we ordered) ←→ Invoice (what vendor says we owe) ←→ Receipt (what we got)

MATCHING RULES:
┌─────────────────────────────────────────────────────────────────┐
│ Match Level    │ Tolerance     │ Action                          │
├────────────────┼───────────────┼─────────────────────────────────┤
│ Perfect match  │ All fields    │ Auto-approve payment            │
│                │ within $0.01  │                                  │
├────────────────┼───────────────┼─────────────────────────────────┤
│ Price variance │ Unit price    │ Flag for buyer review           │
│                │ differs > 2%  │ (price increase without approval)│
├────────────────┼───────────────┼─────────────────────────────────┤
│ Qty variance   │ Received qty  │ Short shipment: adjust payment  │
│                │ < ordered qty │ Over shipment: return or accept  │
├────────────────┼───────────────┼─────────────────────────────────┤
│ No PO match    │ Invoice has   │ Maverick spend: route to manager│
│                │ invalid PO #  │                                  │
├────────────────┼───────────────┼─────────────────────────────────┤
│ Duplicate      │ Same invoice  │ Block payment, flag fraud risk  │
│                │ # + vendor    │                                  │
└─────────────────────────────────────────────────────────────────┘

BUSINESS IMPACT:
- 5% of invoices have discrepancies (at $1B annual spend = $50M at risk)
- Automated detection prevents: overpayments, duplicate payments, fraud
- Average recovery per detected discrepancy: $500-$5,000
- System pays for itself if it catches 100 discrepancies/month
```
