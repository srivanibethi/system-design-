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

### Expected Answers:
- 50K documents/day, growing to 200K
- 60% digital PDF, 30% scanned, 10% email/EDI
- ~500 distinct templates across all customers
- Critical fields: amounts (99%), dates (99%), PO/invoice numbers (99.5%), line items (95%)
- Currently: 40% manual data entry, 60% semi-automated (template matching)
- Multi-tenant: each customer has unique document formats
- Extracted data feeds into ERP and a matching/reconciliation engine
- Processing time: <5 minutes per document
- Learn from corrections within 24 hours

---

## PHASE 2: Requirements & Math (3 minutes)

```
"Let me size this:

THROUGHPUT:
- 200K docs/day (target) = ~2.3 docs/sec sustained
- But bursty: most arrive 8am-6pm business hours = ~5.5 docs/sec
- Peak burst: 10x = 55 docs/sec (month-end invoice dumps)

PROCESSING BUDGET:
- 5 minutes per doc target
- Actual processing time per doc: 10-60 seconds (depending on complexity)
- Bottleneck is ML inference, not I/O

ACCURACY MATH:
- 99% accuracy on amounts means: 1 error per 100 documents
- At 200K docs/day = 2,000 errors/day needing human review
- Human review capacity: 20 reviewers × 100 docs/day = 2,000/day ✓
- Goal: drive auto-approval rate up (currently 60% → target 95%)

STORAGE:
- Avg document: 200KB (PDF) + 5KB (extracted JSON) + 1KB (metadata)
- Daily: 200K × 200KB = 40 GB/day for originals
- Keep originals 7 years (compliance) = ~100 TB
- Extracted data: negligible vs originals

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

## DETAILED SCALE ESTIMATES

### Processing Pipeline Throughput
```
DOCUMENTS: 200K/day target

PROCESSING TIME BUDGET (5 min per doc max):
  But actual processing is much faster:
  - Preprocessing (OCR, layout): 2-5 seconds
  - Classification: 200ms (known template) to 5s (VLM for unknown)
  - Extraction: 1-15 seconds (depends on model tier)
  - Validation: 200ms (rule checks)
  - Total: 3-25 seconds per doc (NOT 5 minutes — that's the SLA)

THROUGHPUT DESIGN:
  200K docs/day ÷ 16 active hours = 12,500 docs/hour = 3.5 docs/sec
  Peak (month-end): 10x = 35 docs/sec
  
  With avg processing time of 10 seconds:
  - Need 35 × 10 = 350 concurrent processing slots at peak
  - If workers have 4 vCPU: 350 ÷ 4 = 88 worker instances (peak)
  - Auto-scale: 20 instances baseline, up to 100 at peak
  
  GPU for VLM/OCR:
  - 20% of docs go to VLM path (40K/day = 2,500/hour = 0.7/sec)
  - VLM inference: 5-15 seconds per doc
  - Need: 0.7 × 10 = 7 concurrent VLM requests
  - 2 GPU instances with batching handle this easily

COST:
┌──────────────────────────────────────────────────────────────────┐
│ Component           │ Cost/doc  │ Daily (200K) │ Monthly         │
├─────────────────────┼───────────┼──────────────┼─────────────────┤
│ OCR (Google Doc AI) │ $0.01     │ $2,000       │ $60K            │
│ Template model      │ $0.001    │ $160 (80%)   │ $5K             │
│ LayoutLM inference  │ $0.005    │ $100 (10%)   │ $3K             │
│ VLM (Claude/GPT-4V) │ $0.05     │ $2,000 (20%) │ $60K            │
│ Compute (workers)   │ $0.002    │ $400         │ $12K            │
│ Storage (S3+DB)     │ $0.0005   │ $100         │ $3K             │
├─────────────────────┼───────────┼──────────────┼─────────────────┤
│ TOTAL               │ ~$0.024   │ ~$4,800      │ ~$143K          │
└──────────────────────────────────────────────────────────────────┘

BREAK-EVEN vs MANUAL:
  Manual data entry: $3-5 per document (offshore), $8-12 (US)
  Our system: $0.024 per doc (automated) + $0.50 per reviewed doc
  Auto-approval rate 95%: avg cost = 0.95×$0.024 + 0.05×$0.55 = $0.05/doc
  Savings: 60-99x cheaper than manual entry
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
