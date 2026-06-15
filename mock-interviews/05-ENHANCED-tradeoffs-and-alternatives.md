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
