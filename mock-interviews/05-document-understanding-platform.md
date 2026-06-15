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
