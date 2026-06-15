# Supply Chain AI Systems Design

## Why This Matters for X

X's moonshot is applying AI to reduce waste, fragility, and inefficiency in global supply chains. The system design interview will likely include at least one problem that touches:
- Enterprise integration (SAP, Oracle ERP)
- Real-time decision making under uncertainty
- AI-augmented human workflows
- Massive-scale optimization problems

---

## Supply Chain System Architecture

### End-to-End AI-Powered Supply Chain Platform
```
┌──────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                  │
├────────────┬──────────────┬──────────────┬───────────────────────┤
│ ERP Systems│ IoT/Sensors  │ External     │ Historical            │
│ (SAP, Oracle)│ (warehouse, │ (weather,    │ (orders, shipments,   │
│            │  transport)  │  market,     │  demand history)      │
│            │              │  suppliers)  │                       │
└─────┬──────┴──────┬───────┴──────┬───────┴───────────┬───────────┘
      │             │              │                   │
      ▼             ▼              ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                   INTEGRATION LAYER                                │
│  CDC (Debezium) │ Event Bus (Kafka) │ API Gateway │ Batch ETL    │
└──────────────────────────────────────┬───────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    AI/ML PLATFORM                                  │
├──────────────┬───────────────┬───────────────┬───────────────────┤
│ Feature Store│ Model Registry│ Training      │ Serving           │
│ (online +    │ (versioned    │ Pipeline      │ Infrastructure    │
│  offline)    │  models)      │ (Vertex AI)   │ (online + batch)  │
└──────────────┴───────────────┴───────────────┴───────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                                 │
├─────────────┬────────────────┬────────────────┬──────────────────┤
│ Demand      │ Inventory      │ Logistics      │ Supplier         │
│ Forecasting │ Optimization   │ Routing        │ Risk             │
│ Service     │ Service        │ Service        │ Assessment       │
└─────────────┴────────────────┴────────────────┴──────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     USER LAYER                                     │
│  Decision Support UI │ Alerts & Recommendations │ Natural Language │
│  (dashboards, what-if)│ (proactive notifications)│ Interface (LLM) │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key System Designs

### 1. Demand Forecasting System

#### Requirements
- Predict demand for 100K+ SKUs across 500+ locations
- Multiple time horizons: next day, next week, next quarter
- Handle seasonality, promotions, external events
- Confidence intervals, not just point estimates
- Explainability for business users

#### Architecture
```
┌─────────────────────────────────────────────────┐
│              DATA COLLECTION                      │
│ Historical sales │ Promotions │ Weather │ Events │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│            FEATURE ENGINEERING                    │
│ Lag features │ Rolling stats │ Calendar features │
│ Price elasticity │ Cannibalization │ External    │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│              MODEL ENSEMBLE                       │
│ ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│ │Statistical│ │ ML-based │ │ Foundation Model │ │
│ │(Prophet,  │ │(LightGBM,│ │ (TimesFM,       │ │
│ │ ETS, ARIMA│ │ XGBoost) │ │  Chronos, Lag-  │ │
│ │           │ │          │ │  Llama)          │ │
│ └──────────┘ └──────────┘ └──────────────────┘ │
│         ↓            ↓              ↓            │
│              ENSEMBLE / META-LEARNER             │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│           SERVING & MONITORING                   │
│ Batch predictions (nightly) + on-demand refresh  │
│ Drift detection │ Accuracy tracking │ Alerts     │
└─────────────────────────────────────────────────┘
```

#### Key Trade-offs
| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Model per SKU vs global | Individual models (accurate) | Global model (generalizes) | Hierarchical: global + SKU-specific adjustments |
| Batch vs streaming | Nightly batch (simpler) | Real-time updates (fresher) | Batch + event-triggered refresh for high-impact SKUs |
| Point vs probabilistic | Single value (simple) | Distribution (actionable) | Quantile regression — always give confidence intervals |

---

### 2. Inventory Optimization Engine

#### The Core Problem
Decide **when** and **how much** to order for each SKU at each location, minimizing:
- Holding costs (capital tied up, warehousing)
- Stockout costs (lost sales, customer churn)
- Ordering costs (per-order fixed costs, bulk discounts)

#### Architecture
```
┌─────────────────────────────────────────────────┐
│              INPUTS                               │
│ Demand forecast │ Lead times │ Costs │ Constraints│
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│         OPTIMIZATION ENGINE                      │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Safety Stock Calculator                    │   │
│  │ SS = z × σ_demand × √(lead_time)         │   │
│  │ + z × demand_avg × σ_lead_time           │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Reorder Point = avg_demand × lead_time    │   │
│  │              + safety_stock               │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │ Order Quantity (EOQ variant)              │   │
│  │ Consider: bulk discounts, shelf life,     │   │
│  │          warehouse capacity, cash flow    │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│        CONSTRAINT SOLVER                         │
│ Budget limits │ Warehouse capacity │ MOQs        │
│ Supplier constraints │ Transportation limits     │
│ → Mixed Integer Programming (OR-Tools / Gurobi) │
└────────────────────────┬────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────┐
│        DECISION SUPPORT                          │
│ Recommended orders │ What-if simulation          │
│ Exception alerts │ Auto-execute below threshold  │
└─────────────────────────────────────────────────┘
```

#### Scaling Challenges
- 100K SKUs × 500 locations = 50M optimization problems
- Interdependencies (substitution, complementary products)
- Multi-echelon (warehouse → DC → store)
- Solution: Decompose into independent subproblems where possible, use heuristics + local optimization

---

### 3. Real-Time Supply Chain Visibility

#### Requirements
- Track shipments across global supply chain in real-time
- Predict delays before they happen (ETA prediction)
- Alert relevant stakeholders automatically
- Handle data from heterogeneous sources (IoT, carrier APIs, customs)

#### Architecture
```
Data Sources                   Processing              Serving
────────────                   ──────────              ───────
Carrier APIs ─┐
GPS/IoT ──────┤               ┌──────────┐
Customs data ─┼──→ Kafka ──→  │  Flink   │──→ Materialized Views
Port systems ─┤               │ (stream  │      (current state)
Weather APIs ─┘               │ process) │          │
                              └──────────┘          ▼
                                   │         ┌──────────────┐
                                   │         │ ETA Prediction│
                                   │         │ Model (ML)    │
                                   │         └──────┬───────┘
                                   │                │
                                   ▼                ▼
                              ┌──────────┐   ┌──────────────┐
                              │ Alert    │   │ Visibility   │
                              │ Engine   │   │ Dashboard    │
                              └──────────┘   └──────────────┘
```

#### ETA Prediction Model
Features:
- Historical transit times for this route/carrier
- Current location and velocity
- Weather conditions along route
- Port congestion data
- Day of week, holidays, peak season

Model: Gradient-boosted trees (LightGBM) or survival analysis

---

## Enterprise Integration Patterns

### ERP Integration (SAP/Oracle)

#### Challenge
Legacy ERP systems are:
- Not cloud-native (often on-premise)
- Batch-oriented (nightly jobs, not real-time)
- Complex schemas (thousands of tables, custom fields)
- Change-resistant (any modification = expensive consulting)

#### Integration Approaches

| Approach | Latency | Complexity | Risk |
|----------|---------|------------|------|
| Direct DB read (CDC) | Seconds | High | Schema coupling |
| API layer (SAP OData, Oracle REST) | Seconds | Medium | Rate limits, auth |
| Middleware (MuleSoft, Dell Boomi) | Seconds-minutes | Low | Vendor lock-in |
| File-based (SFTP/S3) | Hours | Low | Very late data |
| Event-driven (SAP Event Mesh) | Sub-second | Medium | Modern SAP only |

#### Recommended Architecture: Anti-Corruption Layer
```
┌──────────┐     ┌─────────────────┐     ┌──────────────┐
│   ERP    │────→│ Anti-Corruption │────→│  AI Platform │
│ (SAP)    │     │ Layer (ACL)      │     │              │
└──────────┘     └─────────────────┘     └──────────────┘
                        │
                 - Translates ERP schemas to clean domain model
                 - Handles ERP quirks (null encoding, date formats)
                 - Provides stable API even if ERP schema changes
                 - Caches for performance
                 - Validates data quality
```

**Staff insight**: Always propose an Anti-Corruption Layer when integrating with legacy systems. It's a DDD pattern that protects your system from the complexity of the external system.

### Data Quality in Enterprise Context
- **Completeness**: Missing values in ERP exports (30-40% common in enterprise data)
- **Timeliness**: Batch jobs may be delayed; stale data = bad predictions
- **Accuracy**: Manual entry errors, duplicate records
- **Consistency**: Same entity different in different systems (product IDs)

#### Solution: Data Quality Pipeline
```
Raw Data → Schema Validation → Business Rules → Statistical Checks → Clean Data
              (types, nulls)    (valid ranges,    (outlier detection,    (ready for
                                 referential       distribution shift)    ML/analytics)
                                 integrity)
```

---

## Event-Driven Architecture for Supply Chain

### Why Events?
Supply chains are inherently event-driven:
- Order placed, shipped, delivered, returned
- Inventory received, allocated, depleted
- Forecast updated, anomaly detected
- Supplier lead time changed, price changed

### Event Schema Design
```json
{
  "event_id": "evt_abc123",
  "event_type": "inventory.level_changed",
  "timestamp": "2026-06-14T10:30:00Z",
  "source": "warehouse_management_system",
  "entity": {
    "type": "sku_location",
    "id": "SKU-789_WH-SFO"
  },
  "payload": {
    "previous_quantity": 500,
    "new_quantity": 450,
    "change_reason": "order_fulfilled",
    "reference_id": "ORD-456"
  },
  "metadata": {
    "correlation_id": "corr_xyz",
    "schema_version": "2.1"
  }
}
```

### CQRS Pattern (Command Query Responsibility Segregation)
```
Commands (writes)                    Queries (reads)
─────────────────                    ──────────────
Place Order ──→ Command Handler      Dashboard ←── Read Model (materialized view)
                    │                 API ←────── Optimized for queries
                    ▼                              (denormalized, pre-computed)
               Event Store
                    │
                    ▼
              Event Handlers
              (update read models,
               trigger workflows,
               notify services)
```

**When to use CQRS**: Different read and write patterns (supply chain: writes are transactional events, reads are analytical aggregations).

---

## Multi-Tenant Architecture (Enterprise SaaS)

### Isolation Models
| Model | Isolation | Cost | Complexity | Use Case |
|-------|-----------|------|------------|----------|
| Silo (separate DB per tenant) | Highest | Highest | Medium | Regulated industries |
| Bridge (shared DB, separate schemas) | Medium | Medium | Medium | Most enterprise SaaS |
| Pool (shared everything, tenant_id column) | Lowest | Lowest | High (noisy neighbor) | Small tenants, high scale |

### For Supply Chain AI Platform
Recommended: **Bridge model** with per-tenant encryption keys
- Each enterprise customer gets isolated data
- Shared compute infrastructure (cost-efficient)
- Tenant-specific ML models (trained on their data only)
- Cross-tenant models for cold-start (anonymized, federated)

### Noisy Neighbor Prevention
- Resource quotas per tenant (CPU, memory, API calls)
- Separate queues per tenant for batch jobs
- Priority tiers (enterprise → standard → free)
- Circuit breakers to prevent one tenant's load from affecting others
