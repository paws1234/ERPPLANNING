# ERP System Implementation Plan

## 1. Project Overview

**Goal**  
Build a modular, double-entry based ERP platform covering core financial, inventory, supply chain, manufacturing, sales, and human resources operations.

**Architecture Principles**
- Double-entry ledger at the core of all financial postings
- Immutable transaction history
- Multi-company / multi-currency ready
- Hierarchical master data (Chart of Accounts, Locations, BOMs, Org Chart)
- Real-time valuation and stock ledgers
- Workflow-driven approvals and 3-way matching
- Offline-capable POS and biometric attendance integration points
- API-first: one backend API is designed, contract-published and working before the frontend that consumes it; the frontend never reaches the database directly
- One container per service: backend, frontend and database each run as their own container, started together from one compose file (plus a worker container once scheduled jobs exist)
- Separate repositories: backend and frontend are independent repositories — `paws1234/ERPbackend` and `paws1234/ERPfrontend` — sitting inside the planning repository `paws1234/ERPPLANNING`, which holds this plan and gitignores both app folders; each app has its own pipeline and published image, and the published API contract is the only coupling between them

**Technology Stack** (decided 2026-09-17)
- Backend: Python — FastAPI + SQLAlchemy
- Frontend: Next.js
- Database: PostgreSQL
- Job queue: Postgres-backed, no Redis (library pinned when the first scheduled job is built)
- Search: PostgreSQL full-text

**High-Level Modules**
1. Accounting & Financial Management
2. Inventory & Warehouse Management (WMS)
3. Supply Chain & Procurement
4. Manufacturing & Production (MRP / MES)
5. Sales, CRM & Point of Sale (POS)
6. Human Resources & Payroll (HRMS)

---

## 2. Module Breakdown & Scope

### 2.1 Accounting & Financial Management

**Core Components**
- Chart of Accounts (CoA) – tree hierarchy (Assets, Liabilities, Equity, Income, Expense) with localization support
- General Ledger (GL) – immutable transaction ledger (posting date, account, debit, credit, party links)
- Accounts Payable (AP) – supplier invoices, aging, debit notes, payment batches
- Accounts Receivable (AR) – customer invoices, recurring billing, credit limits, dunning, payment gateway webhooks
- Multi-Currency Engine – daily FX rate sync, unrealized/realized gain/loss

**Key Capabilities**
- Automatic double-entry posting from all sub-modules
- Period locking and audit trail
- Financial statements (Trial Balance, P&L, Balance Sheet, Cash Flow)

### 2.2 Inventory & Warehouse Management (WMS)

**Core Components**
- Multi-Warehouse Hierarchy (Warehouse → Zone → Aisle → Bin)
- Stock Ledger & Valuation (FIFO, Moving Average, Standard Cost)
- Item Master (SKU, variants, UOM conversion, barcode/QR)
- Stock Transactions (Receipt, Issue, Transfer, Reconciliation)
- Traceability (Batch/Lot with expiry, Serial numbers)

**Key Capabilities**
- Real-time stock valuation
- Full track-and-trace
- Physical count and adjustment workflows

### 2.3 Supply Chain & Procurement

**Core Components**
- Purchase Requisitions + approval workflows
- Request for Quotation (RFQ) & Supplier Portal
- Purchase Orders & Goods Receipt Notes (GRN)
- 3-way matching (PO ↔ GRN ↔ Supplier Invoice)
- Supplier performance scoring and tax compliance

**Key Capabilities**
- Automated PO generation from RFQ
- Comparative statement matrix
- Vendor scorecards

### 2.4 Manufacturing & Production (MRP / MES)

**Core Components**
- Multi-level Bill of Materials (BOM) with scrap % and routing
- Work Centers & Operations (capacity, hourly rates, downtime)
- Material Requirements Planning (MRP) engine
- Shop Floor Control (Work Orders, Job Cards, Material Issues, Finished Goods)

**Key Capabilities**
- Net requirement calculation (Sales Orders vs Stock)
- Capacity planning and costing
- Time tracking and production output

### 2.5 Sales, CRM & Point of Sale (POS)

**Core Components**
- Lead & Opportunity pipeline (Kanban)
- Sales Orders & Fulfillment (credit checks, pick lists, shipping)
- Pricing Rules & Discounts (customer tiers, volume, campaigns, coupons)
- POS (offline-capable, barcode, cash drawer, Z-Reports)

**Key Capabilities**
- Quotation → Order conversion
- Dynamic pricing engine
- Retail shift management

### 2.6 Human Resources & Payroll (HRMS)

**Core Components**
- Employee master (contracts, assets, org chart)
- Attendance & Shift Management (biometric integration, overtime, late tracking)
- Leave Management (multi-type, accrual, holiday calendar)
- Payroll Engine (attendance linkage, statutory deductions, loans, payslips)

**Key Capabilities**
- End-to-end employee lifecycle
- Automated statutory compliance calculations
- Payslip generation and banking files

---

## 3. Cross-Cutting Concerns

| Concern                    | Approach                                                                 |
|---------------------------|--------------------------------------------------------------------------|
| Double-Entry Integrity    | All modules post balanced journal entries to GL                          |
| Multi-Currency            | Central FX rate service; auto gain/loss calculation                      |
| Audit & Immutability      | Append-only ledgers; soft deletes only on masters                        |
| Workflows & Approvals     | Configurable multi-level approval engine (PR, PO, Leave, etc.)           |
| Localization              | CoA templates, tax rules, statutory reports per country                  |
| Integrations              | Payment gateways, biometric devices, barcode printers, email/SMS         |
| Security                  | Role-based access control (RBAC) + field-level permissions               |
| Reporting                 | Real-time dashboards + scheduled financial & operational reports         |
| Delivery & Packaging      | Separate repositories for backend and frontend; API-first — the published contract is the only coupling between them; one container per service — backend, frontend, database (plus a worker once scheduled jobs exist) — from a single compose file |

---

## 4. Suggested Implementation Phases

### Phase 1 – Foundation (Core Accounting + Inventory)
- Chart of Accounts & General Ledger
- Multi-currency engine
- Item Master, Stock Ledger, basic Warehouse hierarchy
- Stock Receipt / Issue / Transfer
- Batch/Lot (with expiry) and serial tracking on stock movements
- Basic financial reports

**Exit Criteria**:
- Ability to post balanced entries and maintain accurate stock valuation.
- Batch/lot and serial identity is enforced on every movement of a tracked item. (Added 2026-09-17, with the move of traceability from Phase 6 — it changes the shape of the stock ledger, so retrofitting it later would mean migrating live tables.)

### Phase 2 – Procurement & Payables
- Purchase Requisitions → RFQ → PO
- GRN & 3-way matching
- Accounts Payable + payment batches
- Supplier master & performance basics

**Exit Criteria**: End-to-end purchase-to-pay cycle with 3-way match.

### Phase 3 – Sales, Receivables & POS
- Customer master, Credit limits
- Quotations → Sales Orders → Fulfillment
- AR invoicing, dunning, payment webhooks
- Pricing engine
- POS module (online first, offline later)

**Exit Criteria**: Complete order-to-cash cycle including POS sales.

### Phase 4 – Manufacturing
- BOM & Routing
- Work Centers & Operations
- Work Orders + Material Issues + Finished Goods
- Basic MRP run

**Exit Criteria**: Ability to produce finished goods from raw materials with correct costing.

### Phase 5 – HR & Payroll
- Employee master & Org chart
- Attendance & Leave
- Full Payroll engine with statutory rules

**Exit Criteria**: Accurate monthly payroll generation linked to attendance.

### Phase 6 – Advanced Features & Hardening
- Advanced MRP / capacity planning
- Full track-and-trace reporting (forward/backward across production steps)
- Supplier portal
- Offline POS + biometric integration
- Advanced analytics & dashboards
- Performance, security, and localization polish

**Exit Criteria** (added 2026-09-17 — this phase previously stated none):
- Advanced MRP/capacity produces a capacity-respecting, deterministic plan on a constrained dataset
- Forward and backward trace from a batch or serial crosses a production step
- A supplier transacts through the portal with verified isolation from other suppliers
- Offline POS sells without connectivity, synchronises exactly once after a terminal restart, and reports reconciliation differences
- Biometric events ingest without duplication; an unmapped device user is reported
- Dashboards reconcile to their source with one-step drill-down; the scheduled catalogue delivers financial and operational reports
- Performance is measured on the named paths; POS online checkout is within the < 2 s budget at load
- The security review covers every module with no unguarded endpoint and verified isolation; every finding is remediated or an accepted risk is recorded
- Every supported market's localization pack is complete and imports cleanly
- Audit trail coverage is verified for every master and transaction
- The metrics in §6 (1–5) are re-measured and still met after the Phase 6 changes

---

## 5. Data Model Highlights (Core Entities)

- **Account** (hierarchical CoA)
- **Journal Entry / GL Transaction** (immutable)
- **Party** (Customer / Supplier / Employee)
- **Item** + **Item Variant** + **UOM Conversion**
- **Warehouse / Location** (hierarchical)
- **Stock Ledger Entry** (qty + value)
- **Batch / Serial**
- **Purchase Order / Sales Order / Work Order**
- **BOM** + **BOM Item** + **Operation**
- **Employee** + **Attendance Log** + **Leave Application**
- **Payroll Entry** / **Payslip**

---

## 6. Success Metrics

- All financial postings balance (debit = credit)
- Stock valuation matches GL inventory account
- 3-way match rate > 95%
- MRP net requirement accuracy: 100% exact match against independently computed expectations on the verification dataset (any difference is a defect)
- Payroll calculation error rate < 0.1%
- POS transaction latency (online) < 2 s
- Full audit trail for every master and transaction change

---

## 7. Next Steps

1. Technology stack — **decided 2026-09-17**: Python (FastAPI + SQLAlchemy), Next.js, PostgreSQL, Postgres-backed job queue, PostgreSQL full-text search. The queue *library* is the only part still to pin.
2. Define detailed domain models and API contracts for Phase 1.
3. Fix the API conventions — URL versioning, error shape, pagination, auth, idempotency keys on posting endpoints — and publish the generated contract as a **versioned artifact**: with the two applications in separate repositories, that artifact is their only coupling. **API first** (decided 2026-09-17).
4. Package the stack: one container per service — database, backend, frontend (worker added with the first scheduled job) — brought up together by a single compose file that consumes each application's published image, so neither application repository needs its sibling checked out. (Decided 2026-09-17.)
5. Set up CI/CD, testing strategy (unit + integration + ledger integrity tests), building both images.
6. Create localization packs (CoA, tax, statutory) for the **Philippines** (decided 2026-09-17; previously "primary target markets", unnamed).
7. Begin Phase 1 development with GL + Inventory as the backbone.

---

## 8. Decided Configuration Defaults

Defaults the module sections leave open, fixed 2026-09-17. Anything not listed here stays undecided until its phase starts.

| Setting | Decided value |
|---|---|
| Default costing method | Moving Average (FIFO and Standard Cost remain selectable at the configured scope) |
| Valuation scope | Per company |
| Period-lock granularity | Month |
| Negative stock on issue | Never allowed |
| Barcode symbology | EAN/UPC + QR |
| Localization market | Philippines (CoA template, tax rules, statutory deductions and reports, holiday calendar, bank file format) |
| Job queue | Postgres-backed; the library (pgqueuer or procrastinate) is pinned when the first scheduled job is built |

**Still undecided:** fiscal year start (to be fixed with the Philippines pack, not assumed), FX rate source, approval thresholds, 3-way-match tolerance, aging buckets, dunning levels, credit-check mode, MRP horizon and bucket, payroll cutoff, shift and overtime rules, leave accrual rules, supplier scoring weights, report schedules.

**Explicit non-goals** (decided 2026-09-17):
- Inter-company transactions and consolidation — the platform is multi-company by data isolation only
- HR beyond employee master, org chart, attendance, leave and payroll (no recruitment or appraisal)
- Manufacturing beyond BOM, routing, work centres, work orders and MRP (no PLM or quality management)
- CRM beyond the lead/opportunity pipeline and pricing rules (no marketing automation)
- Channels beyond the integrations named in §3 (no storefront, no mobile app other than offline POS)

---

*This plan is derived directly from the provided module specifications and prioritizes a solid double-entry financial core before layering operational modules.*
