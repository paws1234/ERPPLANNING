# Tasks — Derived from plan.md

## Source

- **Plan**: `.github/plan.md` (read-only; this ledger never mutates it)
- **Plan sections ingested**: §1 Overview, §2 Module Breakdown (2.1–2.6), §3 Cross-Cutting Concerns, §4 Phases 1–6, §5 Data Model Highlights, §6 Success Metrics, §7 Next Steps
- **Goal**: Build a modular, double-entry based ERP platform covering core financial, inventory, supply chain, manufacturing, sales, and human resources operations.
- **Constraints** (from §1 Architecture Principles — binding on every task):
  - Double-entry ledger at the core of all financial postings
  - Immutable transaction history
  - Multi-company / multi-currency ready
  - Hierarchical master data (Chart of Accounts, Locations, BOMs, Org Chart)
  - Real-time valuation and stock ledgers
  - Workflow-driven approvals and 3-way matching
  - Offline-capable POS and biometric attendance integration points
- **Cross-cutting approaches** (from §3 — bind all modules):
  - Double-Entry Integrity: all modules post balanced journal entries to GL
  - Multi-Currency: central FX rate service; auto gain/loss calculation
  - Audit & Immutability: append-only ledgers; soft deletes only on masters
  - Workflows & Approvals: configurable multi-level approval engine (PR, PO, Leave, etc.)
  - Localization: CoA templates, tax rules, statutory reports per country
  - Integrations: payment gateways, biometric devices, barcode printers, email/SMS
  - Security: role-based access control (RBAC) + field-level permissions
  - Reporting: real-time dashboards + scheduled financial & operational reports

### Decisions recorded 2026-09-17 (plan amended; this ledger synced)

| Decision | Value |
|---|---|
| Stack | Python + FastAPI + SQLAlchemy · Next.js · PostgreSQL · Postgres-backed job queue (no Redis; library unpinned) · PostgreSQL full-text search |
| Localization market | Philippines |
| Traceability | Batch/lot (with expiry) + serial tracking moved from Phase 6 into Phase 1 — it changes the shape of the stock ledger — as `T-1.INV.08`, `T-1.INV.09`; trace *reporting* stays in Phase 6 |
| Phase 6 exit criteria | Now stated in plan §4 (the phase previously had none) |
| §6 metric 4 (MRP accuracy) | 100 % exact match on the verification dataset; any difference is a defect |
| Phase 1 config defaults | Moving Average · per-company valuation scope · monthly period lock · negative stock never · EAN/UPC + QR barcodes |
| Delivery | **API-first** — the contract is published before the frontend consumes it · **one container per service**: database, backend, frontend (worker added with the first scheduled job) from a single compose file |
| Repositories | **Backend and frontend are separate repositories** — each with its own pipeline and published image; the published API contract artifact is the only coupling between them |
| Confirmed as scheduled | Five §2 components with no phase · capacity split P4/P6 · shared Party in Phase 0 · inter-company out of scope · §7 read as Phase 0 scope |

> **Scope of this ledger**: every task below is `TODO`. No product code, scaffolding, config, or folder is created by `/execute`. Implementation belongs to `/do-task`, one task per run.

---

## Variables & Scope

### Variables

Every value the plan treats as configurable or instance-specific. "Not stated" means the plan does not name a default — that is an Open Question, not a licence to invent one.

| Variable | Meaning / domain | Allowed values / source | Default (per plan) | Applies to |
|---|---|---|---|---|
| `company_id` | Company dimension on every master and posting (multi-company) | Company master (T-0.CORE.03) | 1 company (single-company install) | All modules |
| `fiscal_year_start` | Fiscal calendar per company | Calendar date; from localization pack | Not stated | GL, periods, statements, payroll |
| `base_currency` | Reporting currency per company | Currency master | Not stated | GL, AR, AP, POS, payroll |
| `transaction_currency` | Currency of a document/posting | Currency master (Phase 1) | = `base_currency` | GL, AR, AP, PROC, POS |
| `fx_rate_source` | Provider of daily FX rates | Configured provider | Not stated — "daily FX rate sync" | Multi-currency engine |
| `fx_sync_schedule` | When rates are pulled | Cron expression | Daily | Multi-currency engine |
| `costing_method` | Stock valuation method | `FIFO` \| `Moving Average` \| `Standard Cost` | **Moving Average** (decided 2026-09-17) | Valuation engine, stock ledger, COGS |
| `valuation_scope` | Level at which costing method is chosen | Company \| Item | **Per company** (decided 2026-09-17) | Valuation engine |
| `warehouse_hierarchy_levels` | Depth of location hierarchy | `Warehouse → Zone → Aisle → Bin` (4 levels named) | 4 levels | WMS, stock entries, POS, MRP |
| `uom_conversion_factor` | Conversion between UOMs per item | Per item/UOM pair (≠ 0) | 1 (base UOM) | Item master, all stock transactions |
| `item_variant_attributes` | Attribute set that defines variants | Configured attribute list per item | Empty (no variants) | Item master, pricing, stock ledger |
| `barcode_symbology` | Encoding printed/scanned on items | Barcode \| QR | **EAN/UPC + QR** (decided 2026-09-17) | Item master, POS, printers |
| `allow_negative_stock` | Whether an issue may drive stock below zero | true \| false | **false — never** (decided 2026-09-17) | Stock issues, valuation, POS |
| `traceability_mode` | Track-and-trace level per item | `None` \| `Batch/Lot` \| `Serial` | Not stated | Stock ledger, receipts/issues, Phase 6 traceability |
| `batch_expiry_required` | Expiry enforcement on lots | true \| false | Not stated | Batch/lot handling, FEFO issue |
| `scrap_percent` | Expected scrap per BOM component | 0–100 | Not stated | BOM, MRP net requirements, production costing |
| `operation_sequence` | Ordered operations on a routing | Integer sequence | 1 | Routing, work orders, job cards |
| `work_center_hourly_rate` | Cost rate per work center | Decimal ≥ 0 | Not stated | Work centers, production costing |
| `work_center_capacity` | Available capacity per period | Decimal ≥ 0 + period | Not stated | Work centers, capacity planning |
| `downtime_percent` | Planned/unplanned downtime allowance | 0–100 | Not stated | Capacity planning (basic P4, advanced P6) |
| `mrp_horizon_days` | Planning horizon for net requirements | Integer > 0 | Not stated | MRP engine |
| `mrp_bucket` | Time bucket granularity | Day \| Week | Not stated | MRP engine, planned orders |
| `mrp_demand_sources` | Demand included in the net requirement | Sales Orders \| Sales Orders + others | Sales Orders vs Stock (§2.4) | MRP engine |
| `mrp_accuracy_target` | MRP accuracy success metric | Exact match on the verification dataset | **100 % exact** (decided 2026-09-17; §6 metric 4) | MRP verification, Phase 4 gate |
| `approval_levels` | Depth of approval chain per document type | 1..N per doc type (PR, PO, Payment batch, Leave, …) | Not stated | Approval engine, PROC, AP, LEAVE |
| `approval_thresholds` | Band that selects an approval level | Amount / quantity bands per doc type | Not stated | Approval engine, PROC, AP |
| `period_lock_granularity` | Unit in which periods are locked | Month \| Quarter \| Year | **Month** (decided 2026-09-17) | Period locking, GL |
| `coa_template` | Chart-of-Accounts template per locale/market | From localization pack | **Philippines** pack (decided 2026-09-17) | CoA, GL, statements |
| `tax_pack` | Tax rules per country/document type | From localization pack | **Philippines** pack (decided 2026-09-17) | PROC (supplier tax), AR, AP, PAY |
| `statutory_pack` | Statutory deductions & statutory reports per country | From localization pack | **Philippines** pack (decided 2026-09-17) | Payroll, statutory reports |
| `credit_limit` | Maximum outstanding per customer | Amount ≥ 0 | Not stated | Customer master, order credit check, AR |
| `credit_check_mode` | Behaviour when the limit is breached | `off` \| `warn` \| `block` | Not stated | Sales orders, POS, AR |
| `dunning_levels` | Reminder steps before escalation | N levels with days-past-due + template | Not stated | AR dunning |
| `dunning_channel` | Delivery channel for reminders | `email` \| `SMS` \| `letter` | Not stated | AR dunning, integration boundary |
| `recurring_billing_cycle` | Cadence of recurring invoices | `daily` \| `weekly` \| `monthly` \| custom | Not stated | AR recurring billing |
| `pricing_rule_dimensions` | Inputs a price rule may key on | Customer tier \| Volume \| Campaign \| Coupon | All four named (§2.5) | Pricing engine, sales orders, POS |
| `discount_type` | How a discount is expressed | `percent` \| `amount` | Not stated | Pricing engine, campaigns, coupons |
| `pos_mode` | POS operating mode | `online` \| `offline-capable` | Phase 3 = online first; offline in Phase 6 | POS |
| `payment_tender_types` | Accepted POS tenders | `cash` \| `card` \| `gateway` \| mixed | Not stated | POS tendering, cash drawer, Z-Reports |
| `cash_drawer_required` | Whether a shift needs a cash drawer | true \| false | Not stated | POS shifts |
| `three_way_match_tolerance` | Allowed variance before a match fails | Absolute amount or percent per company | Not stated | 3-way match engine |
| `three_way_match_target` | Match-rate success metric | > 95 % | **> 95 %** (§6) | 3-way match, Phase 2 gate |
| `payment_batch_schedule` | When a payment run executes | Cron / manual | Not stated | AP payment batches |
| `bank_file_format` | Bank transfer file layout | From country/bank pack | **Philippines** pack (decided 2026-09-17) | AP payments, payroll banking files |
| `attendance_source` | Origin of attendance records | `manual` \| `biometric device` | Not stated (device integration in Phase 6) | Attendance log |
| `overtime_rules` | OT multipliers and eligibility thresholds | Rate multiplier + thresholds | Not stated | Attendance, payroll |
| `late_grace_minutes` | Grace before a late mark applies | Integer ≥ 0 | Not stated | Attendance, payroll deductions |
| `shift_definitions` | Shift start/end/break windows | Per shift; overlapping allowed | Not stated | Shift management, attendance, payroll |
| `roster_cycle` | Pattern in which shifts repeat | Daily \| Weekly \| Custom cycle | Not stated | Shift management |
| `leave_types` | Leave categories available to employees | Configured list (multi-type) | Not stated | Leave management, payroll |
| `leave_accrual_rule` | How balances accrue | Monthly \| Annual \| Per-cycle + carry-forward cap | Not stated | Leave management |
| `holiday_calendar` | Non-working days per country/region/year | From localization pack | **Philippines** pack (decided 2026-09-17) | Leave, attendance, payroll |
| `payroll_cycle` | Payroll period | Monthly (§4 Phase 5 says monthly) | Monthly | Payroll engine |
| `payroll_cutoff_day` | Attendance/leave cutoff feeding a run | Day of month | Not stated | Payroll engine |
| `payroll_components` | Earnings/deduction building blocks | Configured per company | Not stated | Payroll structure, payslips |
| `loan_installment_policy` | Recovery limits and order for loans/advances | Max % of net pay; priority order | Not stated | Loans & advances, payroll |
| `payroll_error_tolerance` | Accuracy metric for payroll | < 0.1 % | **< 0.1 %** (§6) | Payroll engine, Phase 5 gate |
| `pos_latency_budget` | Online POS transaction latency | < 2 s | **< 2 s** (§6) | POS, Phase 3 gate |
| `rbac_roles` | Role set used for authorisation | Role list per deployment | Not stated | All modules (T-0.SEC.01) |
| `field_level_permissions` | Read/write rights per role × field | Matrix | Not stated | All modules (T-0.SEC.01) |
| `integration_endpoints` | Credentials/URLs for gateway, biometric device, barcode printer, email/SMS | Configured per tenant | Not stated | Integration boundary, AR, POS, HR, dunning |
| `report_schedule` | When a report is generated and delivered | Cron + recipient list | Not stated | Reporting framework, Phase 6 analytics |
| `job_queue` | Background/scheduled job runner (no Redis) | `pgqueuer` \| `procrastinate` | **Postgres-backed; library unpinned** (decided 2026-09-17) | Reporting runner, dunning, recurring billing, offline POS sync, biometric pulls |
| `api_base_url` | The frontend's backend endpoint, injected at container start | URL from the environment | Not stated — supplied per environment | Frontend container, API client |
| `cors_allowed_origins` | Origins permitted to call the API | List of origins | Frontend origin only | API boundary |
| `container_registry` | Where the two service images are published and pulled from | Registry URL + namespace | Not stated | `T-0.CICD.01`, `T-0.CICD.02`, `T-0.DEPLOY.03` |
| `journal_balance_invariant` | Debit must equal credit on every posting | Always on — **not configurable** | Always on | All posting paths (§6 metric 1) |

### Success metrics → owning tasks

| Metric (§6) | Owning task(s) | Gate |
|---|---|---|
| All financial postings balance (debit = credit) | T-0.CORE.01, T-0.CORE.02, T-1.ACCT.03 | T-1.X.GATE |
| Stock valuation matches GL inventory account | T-1.INV.07 | T-1.X.GATE |
| 3-way match rate > 95 % | T-2.MATCH.01, T-2.MATCH.02 | T-2.X.GATE |
| MRP net requirement accuracy — **100 % exact on the verification dataset** (decided 2026-09-17) | T-4.MRP.03 | T-4.X.GATE |
| Payroll calculation error rate < 0.1 % | T-5.PAY.08 | T-5.X.GATE |
| POS transaction latency (online) < 2 s | T-3.POS.06 | T-3.X.GATE |
| Full audit trail for every master and transaction change | T-0.AUDIT.02, T-6.HARD.04 | T-6.X.GATE |

### Global scope

- **IN** — the six modules of §1 (Accounting & Financial Management, Inventory & Warehouse Management, Supply Chain & Procurement, Manufacturing & Production, Sales/CRM/POS, HR & Payroll), the eight cross-cutting approaches of §3, the core entities of §5, the phases 1–6 of §4 (plus the Phase 0 foundations derived from §3 and §7), and the metrics of §6.
- **OUT** — anything §4 assigns to a later phase (see phase scopes below); this ledger is the only artifact `/execute` writes.
- **NON-GOALS** — the plan names no explicit non-goals. Out of scope by omission, and *not* to be added by any `/do-task` run:
  - HR functionality beyond §2.6 (e.g. recruitment, performance appraisal) — not named
  - Manufacturing beyond §2.4 (e.g. PLM, quality management) — not named
  - CRM beyond the lead/opportunity pipeline and pricing rules — not named
  - Channels beyond the integrations listed in §3 (no e-commerce storefront, no mobile app other than offline POS) — not named
  - Markets/locales beyond the localization packs — not named
  - Technology choices beyond §7.1 — the plan leaves the stack open, so it is a task (T-0.STACK.01), not an assumption

### Phase scopes

**Phase 0 — Cross-cutting foundations** (derived: §3 + §7.1–§7.6; the plan defines no Phase 0 of its own)
- **IN**: stack decision, CI/CD, testing + ledger-integrity strategy, double-entry posting primitive, company/fiscal master, append-only + audit conventions, approval engine, RBAC, shared Party master, integration boundary, reporting framework, localization packs, Phase 1 domain models & API contracts, API conventions + published contract, frontend shell + typed client, one container per service, and the backend/frontend split into two repositories
- **OUT**: any module-specific business logic (all of it is Phase 1+) and production orchestration beyond the single compose stack (see Open Questions)
- **EXIT CRITERIA**: none stated in the plan → derived from §3 and §7 and checked in `T-0.X.GATE`

**Phase 1 — Foundation (Core Accounting + Inventory)**
- **IN**: Chart of Accounts & General Ledger; multi-currency engine; Item Master, Stock Ledger, basic Warehouse hierarchy; Stock Receipt / Issue / Transfer; Batch/Lot (with expiry) and serial tracking (moved here from Phase 6 on 2026-09-17); Basic financial reports
- **OUT**: AP, AR, procurement, sales, manufacturing, HR (Phases 2–5); offline POS and biometric integration, and forward/backward trace *reporting* (Phase 6)
- **EXIT CRITERIA** (plan §4, amended 2026-09-17): *Ability to post balanced entries and maintain accurate stock valuation.* · *Batch/lot and serial identity is enforced on every movement of a tracked item.*

**Phase 2 — Procurement & Payables**
- **IN**: Purchase Requisitions → RFQ → PO; GRN & 3-way matching; Accounts Payable + payment batches; Supplier master & performance basics
- **OUT**: sales, POS, manufacturing, HR; supplier portal and advanced supplier scoring (Phase 6)
- **EXIT CRITERIA** (verbatim): *End-to-end purchase-to-pay cycle with 3-way match.*

**Phase 3 — Sales, Receivables & POS**
- **IN**: Customer master, Credit limits; Quotations → Sales Orders → Fulfillment; AR invoicing, dunning, payment webhooks; Pricing engine; POS module (online first, offline later)
- **OUT**: offline POS (Phase 6), manufacturing, HR; supplier portal (Phase 6)
- **EXIT CRITERIA** (verbatim): *Complete order-to-cash cycle including POS sales.*

**Phase 4 — Manufacturing**
- **IN**: BOM & Routing; Work Centers & Operations; Work Orders + Material Issues + Finished Goods; Basic MRP run
- **OUT**: advanced MRP / capacity planning and batch-serial traceability (Phase 6)
- **EXIT CRITERIA** (verbatim): *Ability to produce finished goods from raw materials with correct costing.*

**Phase 5 — HR & Payroll**
- **IN**: Employee master & Org chart; Attendance & Leave; Full Payroll engine with statutory rules
- **OUT**: biometric device integration (Phase 6); payroll for markets without a localization pack
- **EXIT CRITERIA** (verbatim): *Accurate monthly payroll generation linked to attendance.*

**Phase 6 — Advanced Features & Hardening**
- **IN**: Advanced MRP / capacity planning; Full track-and-trace reporting (forward/backward across production); Supplier portal; Offline POS + biometric integration; Advanced analytics & dashboards; Performance, security, and localization polish
- **OUT**: nothing new — this phase hardens and completes what Phases 1–5 built; batch/lot and serial *tracking* now lives in Phase 1
- **EXIT CRITERIA** (added to plan §4 on 2026-09-17 — previously unstated; restated as checks in `T-6.X.GATE`): advanced plan is capacity-respecting and deterministic · forward/backward trace crosses a production step · portal isolation verified · offline POS syncs exactly once · biometric ingest idempotent · dashboards and scheduled reports reconcile · performance measured with POS < 2 s at load · security review complete with no unguarded endpoint · every market pack complete · audit coverage complete · §6 metrics 1–5 re-measured and still met

---

## Task ID scheme

`T-<Phase>.<Stream>.<Seq>`

- **Phase** — `0` = cross-cutting foundations (derived from §3 and §7, which §4 does not schedule); `1`–`6` = exactly the phases in §4.
- **Stream** — a short code derived from the plan's own module/workstream names:
  - Phase 0: `STACK`, `CORE`, `MODELS`, `API`, `AUDIT`, `PARTY`, `WF`, `SEC`, `INT`, `REPORT`, `LOC`, `DEPLOY`, `CICD`
  - Phase 1: `ACCT` (§2.1), `INV` (§2.2 — including batch/lot and serial tracking since 2026-09-17)
  - Phase 2: `PROC`, `AP` (§2.3, §2.1), `MATCH` (§2.3 3-way matching)
  - Phase 3: `SALES` (§2.5), `AR` (§2.1), `POS` (§2.5)
  - Phase 4: `BOM`, `WC`, `WO`, `MRP` (§2.4)
  - Phase 5: `EMP`, `ATT`, `LEAVE`, `PAY` (§2.6)
  - Phase 6: `ADV`, `TRACE` (trace *reporting* only — tracking is Phase 1), `PORTAL`, `OFFLINE`, `ANALYTICS`, `HARD` (§4 Phase 6)
- **Seq** — zero-padded sequence within the stream.
- **GATE** — `T-<Phase>.X.GATE`, one per phase; must be `DONE` before any task of a later phase may start.
- **Ordering (API-first, decided 2026-09-17)** — within every phase, backend/API tasks precede the frontend tasks that consume them: no UI task starts before the API it reads is contract-published. The frontend is a separate deployable and never reaches the database.

---

## Repositories (decided 2026-09-17)

The backend and the frontend live in **separate repositories**. Each has its own pipeline, its own published image and its own release cadence. The published API contract artifact is the only coupling: no shared source tree, no cross-repo build, and no repository needing its sibling checked out.

### Created 2026-09-17

| Repo | Local path (relative to the workspace root) | Remote | State |
|---|---|---|---|
| Planning (this one) | `ERPV1/.` | `https://github.com/paws1234/ERPPLANNING.git` | initialised on `main`, `origin` set, **no commits yet**; tracks `.github/` only |
| Backend | `ERPV1/ERPbackend` | `https://github.com/paws1234/ERPbackend.git` | initialised on `main`, `origin` set, **no commits yet** |
| Frontend | `ERPV1/ERPfrontend` | `https://github.com/paws1234/ERPfrontend.git` | initialised on `main`, `origin` set, **no commits yet** |

All three remotes are **public** and empty. `ERPbackend/` and `ERPfrontend/` are **gitignored** in the planning repo — `git check-ignore` confirms both — so they are never committed here, not even as gitlinks. The apps' first commits belong to the Phase 0 packaging tasks (`T-0.DEPLOY.01`, `T-0.DEPLOY.02`). Being public, no `.env`, credential or secret may ever be committed to any of the three.

The planning repo tracks `.github/` (this ledger, the plan, the skills) plus the root `.gitignore`. A `/do-task` run starts from the workspace root and lands its diff in the subfolder named in the map below.

The skills carry this map: `/plan` records the repository layout in the plan, `/execute` emits this `## Repositories` section into the ledger, and `/do-task` reads it before writing — so a run routes backend/API work to `ERPbackend` and frontend/UI work to `ERPfrontend` rather than guessing.

| Repo | Tasks |
|---|---|
| **Backend repository** | every task not listed below |
| **Frontend repository** | `T-0.API.02`, `T-0.DEPLOY.02`, `T-0.CICD.02`, `T-2.PROC.04`, `T-3.SALES.02`, `T-5.EMP.02`, `T-6.ANALYTICS.01` |
| **Both** — client in the frontend repo, API and postings in the backend repo | `T-3.POS.01`, `T-3.POS.02`, `T-3.POS.03`, `T-3.POS.04`, `T-6.PORTAL.01`, `T-6.OFFLINE.01`; the verification tasks (`T-0.X.GATE` … `T-6.X.GATE`, `T-6.HARD.01`, `T-6.HARD.02`, `T-6.HARD.04`) span both repos |
| **Deployment stack** — `T-0.DEPLOY.03` | repository undecided; see Open Questions |

A `/do-task` run must land its diff in the repository named in the map above — `ERPV1/ERPbackend` or `ERPV1/ERPfrontend` — running from the workspace root.

---

## Cross-cutting / foundations

> Derived from §3 (Cross-Cutting Concerns) and §7 (Next Steps). §4 does not schedule these, but every phase depends on them, so they form Phase 0.

### Stream: `STACK`

#### Task ID: T-0.STACK.01
- **Title**: Finalize the technology stack
- **Description**: Decide and record backend, frontend, database, queue, and search choices. Produce a single decision record naming each choice and the reason.
- **Plan source**: §7.1
- **Scope**: Decision record only. No project scaffolding, no dependencies installed, no code.
- **Variables / Config**: none read; this decision fixes the storage/runtime every other task targets.
- **Decision (user, 2026-09-17)**: backend = **Python + FastAPI + SQLAlchemy**; frontend = **Next.js**; database = **PostgreSQL**; queue = **Postgres-backed, no Redis** (library unpinned); search = **PostgreSQL full-text**. Recorded in plan §1 and §8.
- **Dependencies**: none
- **Acceptance Criteria**: One written decision stating all five choices and the reason for each, justified against the §3 cross-cutting needs (immutable append-only ledgers, multi-currency, offline POS sync, scheduled reporting); the only sub-choice it may leave open is which Postgres job-queue library (pgqueuer or procrastinate) is used.
- **Evidence**: `ERPbackend/TECH-STACK.md` (backend repository) — five choices, a reason per choice, the §3 justification table, consequences; the queue library left open as permitted, to be pinned in T-0.REPORT.01. Referenced by T-0.MODELS.01 and T-0.CICD.01. No runnable check: the deliverable is a written decision, not logic.
- **Estimated Effort**: S
- **Owner Role**: Tech Lead / Architect
- **Status**: DONE

### Stream: `CORE`

#### Task ID: T-0.CORE.01
- **Title**: Double-entry posting primitive (balanced journal posting)
- **Description**: Implement the single shared posting primitive every module calls: it accepts journal lines (posting date, account, debit, credit, party links) and refuses to persist anything unless total debit equals total credit, with at least two lines.
- **Plan source**: §1 principle 1, §3 row "Double-Entry Integrity", §5 Journal Entry / GL Transaction
- **Scope**: The invariant and the posting API only. No accounts UI, no statements, no reporting.
- **Variables / Config**: `journal_balance_invariant` (always on); `base_currency`, `company_id` on each entry.
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: An unbalanced set of lines is rejected with a clear error and nothing persisted; a balanced set persists atomically; there is exactly one posting entry point used by all callers (no module-local posting code); the check is enforced at the storage boundary, not only in the caller.
- **Evidence**: `ERPbackend/app/ledger/posting.py` (the primitive) and `ERPbackend/app/db.py` (shared declarative base), plus the one check `ERPbackend/tests/check_posting_invariant.py` — run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_posting_invariant.py`), all four assertions green: a balanced set persists as one unit; an **unbalanced** set is refused by the primitive with `entry does not balance: debit 100.00 != credit 90.00` and leaves no row behind; a **single-line** set is refused with `a posting needs at least 2 lines, got 1`; and **raw SQL that bypasses the primitive** is refused at COMMIT by the database's own DEFERRABLE INITIALLY DEFERRED constraint triggers (`does not balance`, `has 0 line(s); a posting needs at least 2`) — the storage boundary, not only the caller. `post_journal_entry` is the only writer of `journal_entry` / `journal_line` in the tree. The whole-ledger scan and the CI wiring are T-0.CORE.02 and T-0.CICD.01.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-0.CORE.02
- **Title**: Testing strategy and ledger-integrity test harness
- **Description**: Define the unit / integration / ledger-integrity test layers, and build the harness that asserts metric 1 of §6 across the whole ledger: every persisted journal entry balances, and no posting path bypasses the primitive.
- **Plan source**: §7.5, §6 metric 1
- **Scope**: The test strategy and the integrity assertions. Not a general test framework rollout for every module's own tests.
- **Variables / Config**: none beyond the posting primitive's contract.
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: Strategy names the three layers with what belongs in each; a runnable check fails when a deliberately unbalanced entry is injected; the check scans the stored ledger (not just the API responses) and reports any imbalance.
- **Evidence**: `ERPbackend/TESTING.md` (the strategy: the three layers and what belongs in each) plus the harness `ERPbackend/tests/check_ledger_integrity.py` — `ledger_gate(connection)` scans the **stored** `journal_entry` / `journal_line` rows, never API responses, and returns `1` after reporting every entry that does not balance or has fewer than two lines. Run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ledger_integrity.py`, exit 0): **green on a clean ledger** (one real posting through `post_journal_entry`, gate returns 0); **red on a deliberately injected** entry — reported as `2 line(s), debit 10.000000 <> credit 9.000000` — and on an entry with no lines (`0 line(s)`), each injected with the tables' `TRIGGER USER` disabled (the way a restored dump or hand-edit arrives) and rolled back so nothing survives; the run fails if either injection is not reported. Wiring the layers into a pipeline is T-0.CICD.01 — no CI exists yet, so the evidence is the executed check.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: DONE

#### Task ID: T-0.CORE.03
- **Title**: Company and fiscal calendar master (multi-company scoping)
- **Description**: Model the owning company and its fiscal calendar, and apply the company dimension to all master data and postings so the platform is multi-company from the first table.
- **Plan source**: §1 principle 3, §2.1 (CoA localization support), §3 row "Localization"
- **Scope**: Company entity, fiscal calendar, and the scoping convention. No inter-company transactions or consolidation (§4 never names them).
- **Variables / Config**: `company_id`, `fiscal_year_start`, `base_currency`.
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: A second company can be created without schema change and cannot see the first company's data; fiscal calendar is per company; every master and posting table carries the company dimension.
- **Evidence**: `ERPbackend/app/company.py` — the `Company` master (unique `code`, `base_currency`, `fiscal_year_start_month` 1–12, **no default**: plan §8 leaves the Philippines' fiscal year start undecided, so creating a company must state it). `ERPbackend/app/db.py` — the scoping convention, registered on the shared metadata so no model module can build a schema without it: a table is company-scoped when it has `company_id`, or declared in `GLOBAL_TABLES`/`CHILD_TABLES` (reason per entry), and anything else **fails the build** (`UnscopedTableError`); creating the schema also turns on row-level security on every scoped table, one policy comparing `company_id` with the session's `app.company_id`, with `scope_to_company(session, id)` binding a transaction to one company (transaction-scoped, so no caller has to remember a filter). `ERPbackend/app/ledger/posting.py` — `journal_entry.company_id` is now the foreign key `→ company.id`. Check: `ERPbackend/tests/check_company_isolation.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_company_isolation.py`, exit 0) connected as a **plain non-owner role** (`SET ROLE`; a superuser bypasses RLS, which would prove nothing) — green on all four: a second company created on the schema already in place, each keeping its own fiscal calendar; the guard refusing an unscoped table (`table 'unscoped_probe' has no company_id and is declared neither global nor a child…`); alpha's posting invisible to beta for **entry and lines**, and beta refused a row for alpha (`new row violates row-level security policy for table "journal_entry"`); an unscoped session seeing nothing. Sensitivity shown separately: with `DISABLE ROW LEVEL SECURITY` the same query returns beta 1 row instead of 0. The two earlier checks were re-run green on the same database after the foreign key landed (`check_posting_invariant.py`, `check_ledger_integrity.py`), each creating the company it posts for.
- **Estimated Effort**: M
- **Owner Role**: Database Engineer
- **Status**: DONE

### Stream: `MODELS`

#### Task ID: T-0.MODELS.01
- **Title**: Detailed Phase 1 domain models and API contracts
- **Description**: Specify the Phase 1 entities and their API contracts in detail: Account, Journal Entry / GL Transaction, Item, Item Variant, UOM Conversion, Warehouse/Location, Stock Ledger Entry.
- **Plan source**: §5 (core entities), §7.2
- **Scope**: Phase 1 entities only. Not Phase 2–6 entities, not UI design.
- **Variables / Config**: `warehouse_hierarchy_levels`, `uom_conversion_factor`, `item_variant_attributes`, `costing_method`, `base_currency`, `fx_rate_source`.
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: Each §5 Phase-1 entity has fields, relationships, cardinality and an input/output contract; posting date, account, debit, credit and party links are represented exactly as §2.1 states; hierarchy and UOM conversion semantics are unambiguous; no entity beyond Phase 1 is defined.
- **Evidence**: The contract document, used unmodified by T-1.ACCT.* and T-1.INV.*.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Accounting)
- **Status**: TODO

### Stream: `API`

#### Task ID: T-0.API.01
- **Title**: API conventions and contract publication
- **Description**: Fix the conventions every endpoint follows — URL versioning, one error shape, pagination and filtering, auth claims, and idempotency keys on posting endpoints — and publish the generated OpenAPI contract as a **versioned artifact** that the frontend repository consumes without access to this repository's source.
- **Plan source**: §1 principle "API-first", §3 row "Delivery & Packaging", §7.3
- **Scope**: Cross-cutting conventions, contract publication and the version policy. Per-entity contracts are T-0.MODELS.01 and the phases' own tasks.
- **Variables / Config**: `api_base_url`, `cors_allowed_origins`, `company_id` (how it is supplied on a request), `rbac_roles` (auth claims).
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: The contract is generated from the code rather than hand-maintained, is fetchable from a running container, and is published as a versioned artifact the frontend repository can pull without this repository's source; a breaking change requires a new version instead of mutating the published one; every endpoint returns errors in one shape; a posting endpoint honours an idempotency key so a retried request does not post twice; every documented field states its type and whether it is required.
- **Evidence**: The published contract plus one endpoint exercised for each convention (error shape, pagination, idempotent retry).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-0.API.02
- **Title**: Frontend application shell and typed API client
- **Description**: Scaffold the Next.js application in the **frontend repository** as a separate deployable: routing and layout shell, session/auth handling against the API, and a typed client generated from the published contract artifact.
- **Plan source**: §1 principles "API-first", "one container per service" and "separate repositories", §7.3, §7.4
- **Scope**: The shell, the session and the client, in the frontend repository. Feature screens belong to their phases.
- **Variables / Config**: `api_base_url`, `cors_allowed_origins`, `rbac_roles` (what the shell reveals).
- **Dependencies**: T-0.API.01, T-0.SEC.01
- **Acceptance Criteria**: The frontend's calls go through the generated client and it holds no database credentials or direct database connection; the repository builds and typechecks from the published contract artifact alone, with no backend checkout present; a 401 leads into a session flow and a 403 does not break the shell; a contract field removed breaks the build rather than failing at runtime; the app runs as its own process, not embedded in the backend.
- **Evidence**: The shell running as a separate process against the containerised API, with one authorised and one denied call shown.
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: TODO

### Stream: `AUDIT`

#### Task ID: T-0.AUDIT.01
- **Title**: Append-only history and soft-delete convention
- **Description**: Implement the storage convention that ledgers and stock/GL entries are append-only (corrections are new entries, never updates or deletes), while masters use soft deletes only.
- **Plan source**: §1 principle 2, §3 row "Audit & Immutability"
- **Scope**: The convention and its enforcement. No audit UI, no reporting.
- **Variables / Config**: none (not configurable — the principle is unconditional).
- **Dependencies**: T-0.STACK.01, T-0.CORE.03
- **Acceptance Criteria**: An update or delete against a posted ledger/stock entry is rejected at the storage boundary; master records mark deleted state without removing the row; existing reads exclude soft-deleted masters by default.
- **Evidence**: A rejection check for update/delete plus a soft-delete/read-filter check, both in CI.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-0.AUDIT.02
- **Title**: Audit trail framework (who / when / what, every master and transaction)
- **Description**: Record actor, timestamp, before/after values and origin for every master change and every transaction change, so metric 7 of §6 holds across all six modules.
- **Plan source**: §3 row "Audit & Immutability", §6 metric 7
- **Scope**: Capture and read-back of the trail. No audit dashboards (Phase 6), no log shipping.
- **Variables / Config**: `rbac_roles` (actor identity), `company_id`.
- **Dependencies**: T-0.AUDIT.01
- **Acceptance Criteria**: Creating, editing and soft-deleting a master produces a trail row with actor, timestamp and changed values; a posted transaction's creation is traceable to its originating document; the trail cannot be edited through the application's own API.
- **Evidence**: A representative master + transaction exercise whose read-back shows both events.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `PARTY`

#### Task ID: T-0.PARTY.01
- **Title**: Shared Party master (Customer / Supplier / Employee)
- **Description**: Model one party identity with the role(s) a party holds, so Supplier (Phase 2), Customer (Phase 3) and Employee (Phase 5) share one record rather than three parallel masters.
- **Plan source**: §5 Party (Customer / Supplier / Employee), §2.2 hierarchical master data principle
- **Scope**: The base party and its roles only. Role-specific attributes (credit limits, bank details, contracts) belong to the owning modules.
- **Variables / Config**: `company_id`; tax identifiers (consumed by `tax_pack` in Phase 2).
- **Dependencies**: T-0.STACK.01, T-0.AUDIT.01
- **Acceptance Criteria**: One party can hold several roles without duplication; role-specific data is not stored on the base record; a party with no role is rejected; parties are soft-deletable, never hard-deleted.
- **Evidence**: A check that links one party to two roles and fails if a second master row is required.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `WF`

#### Task ID: T-0.WF.01
- **Title**: Configurable multi-level approval engine
- **Description**: Build the workflow engine that any document type can attach to: ordered approval levels, threshold bands that route a document, approve/reject/return transitions, and a full decision history.
- **Plan source**: §3 row "Workflows & Approvals", §1 principle 6
- **Scope**: The engine and its configuration surface, exercised by PR/PO/payment/leave in later phases. No document-specific approval rules here.
- **Variables / Config**: `approval_levels`, `approval_thresholds`, `rbac_roles`.
- **Dependencies**: T-0.STACK.01, T-0.AUDIT.02
- **Acceptance Criteria**: A document type is configured with levels and thresholds without code change; a document follows exactly the configured chain; rejection and return are recorded with actor and reason; approval history is immutable.
- **Evidence**: One configured two-level chain driven end to end, with the decision history shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `SEC`

#### Task ID: T-0.SEC.01
- **Title**: Role-based access control with field-level permissions
- **Description**: Implement roles, permissions, and field-level read/write restrictions, applied uniformly across modules.
- **Plan source**: §3 row "Security"
- **Scope**: Authorisation only. No authentication provider choice beyond what T-0.STACK.01 fixed, no SSO features the plan does not name.
- **Variables / Config**: `rbac_roles`, `field_level_permissions`.
- **Dependencies**: T-0.STACK.01, T-0.AUDIT.02
- **Acceptance Criteria**: A role without a permission is refused at the API boundary (not merely hidden in the UI); a field marked restricted is absent from both read payloads and write acceptance; permission changes take effect without a deploy; every refusal is attributable in the audit trail.
- **Evidence**: A check driving one allowed and one denied path per level (endpoint and field).
- **Estimated Effort**: M
- **Owner Role**: Security Engineer
- **Status**: TODO

### Stream: `INT`

#### Task ID: T-0.INT.01
- **Title**: Integration boundary: outbound adapters and inbound webhook receiver
- **Description**: Establish the single boundary through which the platform talks to externals named in §3: payment gateways (inbound webhooks), biometric devices (inbound sync), barcode printers, and email/SMS (outbound). Includes retry, idempotency keys, and a delivery log.
- **Plan source**: §3 row "Integrations"
- **Scope**: The boundary, contracts and delivery log. The gateway-specific, device-specific and printer-specific mappings belong to the phases that use them.
- **Variables / Config**: `integration_endpoints`, `dunning_channel` (as an outbound consumer), `barcode_symbology` (printer consumer).
- **Dependencies**: T-0.STACK.01, T-0.SEC.01
- **Acceptance Criteria**: A duplicated inbound webhook (same idempotency key) is processed once; a failing outbound send is retried and visible in the delivery log; credentials are configuration, not code; a consumer cannot bypass the boundary.
- **Evidence**: A duplicated delivery check and a failed-then-retried send, both observed in the delivery log.
- **Estimated Effort**: M
- **Owner Role**: Integration Engineer
- **Status**: TODO

### Stream: `REPORT`

#### Task ID: T-0.REPORT.01
- **Title**: Reporting framework (real-time query layer + scheduled runner)
- **Description**: Provide the read layer used by real-time dashboards and the runner that executes scheduled financial and operational reports, honouring `report_schedule` and permissions.
- **Plan source**: §3 row "Reporting"
- **Scope**: Framework, scheduling and delivery only. The report content itself is built in Phase 1 (financial statements) and Phase 6 (catalogue/dashboards).
- **Variables / Config**: `report_schedule`, `rbac_roles`, `company_id`.
- **Dependencies**: T-0.STACK.01, T-0.SEC.01
- **Acceptance Criteria**: A registered report is scheduled, produced on time, and delivered to its recipients; a report run is scoped to the requester's company and permissions; a failed run is visible rather than silent.
- **Evidence**: One scheduled report produced and delivered (or its delivery logged) with a failed run also shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `LOC`

#### Task ID: T-0.LOC.01
- **Title**: Localization pack framework and the Philippines pack (CoA templates, tax rules, statutory reports)
- **Description**: Define the pack structure for CoA templates, tax rules and statutory reports, then author the pack for the confirmed market — the **Philippines** (decided 2026-09-17; §7.4 previously left the market unnamed, and plan §8 now records it).
- **Plan source**: §3 row "Localization", §2.1 (CoA localization support), §2.3 (tax compliance), §2.6 (statutory compliance), §7.6, §8
- **Scope**: Pack structure plus one complete pack per confirmed market: CoA template, tax rules, statutory deduction rules, statutory report definitions, holiday calendar, fiscal calendar, bank file format. No module logic.
- **Variables / Config**: `coa_template`, `tax_pack`, `statutory_pack`, `holiday_calendar`, `fiscal_year_start`, `bank_file_format`.
- **Dependencies**: T-0.STACK.01, T-0.CORE.03
- **Acceptance Criteria**: Pack structure is documented and versioned; the Philippines pack has a CoA template that imports cleanly, tax rules covering the documents that market needs, statutory deduction rules, its statutory report definitions, a holiday calendar and its bank file format; the fiscal year start is confirmed with the pack — it is still undecided as of 2026-09-17 and must not be assumed; no market beyond the Philippines is invented.
- **Evidence**: The Philippines pack imported into a test company and validated; the pack version and the confirmed fiscal year start stored with it.
- **Estimated Effort**: L
- **Owner Role**: Domain Analyst (Payroll/Statutory) + Domain Analyst (Accounting)
- **Status**: TODO

### Stream: `DEPLOY`

#### Task ID: T-0.DEPLOY.01
- **Title**: Backend container image
- **Description**: Package the FastAPI application as a buildable image: pinned dependencies, non-root runtime user, configuration entirely from environment variables, no secrets baked in, and a start command that needs no shell wrapper (so signals reach the process).
- **Plan source**: §1 principle "one container per service", §3 row "Delivery & Packaging", §7.4
- **Scope**: The backend image in the backend repository. The database and frontend images are separate tasks.
- **Variables / Config**: database connection settings and `company_id` defaults, supplied by the environment; no secret is read from a file inside the image.
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: The image builds from a clean checkout and starts against a database given only environment variables; it runs as a non-root user; no secret, credential or `.env` file is copied into the image; the app process is PID 1 and stops on SIGTERM without being killed after a timeout; the image is reproducible from a pinned dependency lock.
- **Evidence**: A build followed by a start against the compose database, the image inspected for baked secrets, and the observed stop behaviour.
- **Estimated Effort**: S
- **Owner Role**: DevOps Engineer
- **Status**: TODO

#### Task ID: T-0.DEPLOY.02
- **Title**: Frontend container image
- **Description**: Package the Next.js application as its own image: production build, API base URL injected from the environment at start, and a runtime that does not carry the build toolchain.
- **Plan source**: §1 principle "one container per service", §3 row "Delivery & Packaging", §7.4
- **Scope**: The frontend image in the frontend repository. The backend image is T-0.DEPLOY.01; the database runs the stock PostgreSQL image.
- **Variables / Config**: `api_base_url`, `cors_allowed_origins`.
- **Dependencies**: T-0.DEPLOY.01, T-0.API.02
- **Acceptance Criteria**: The image builds from a clean checkout and serves the app; the API base URL is read from the environment at start, so one image works against a different backend without a rebuild; source-only toolchain and development dependencies are absent from the runtime image; it runs as a non-root user and stops on SIGTERM.
- **Evidence**: A build followed by a start against the containerised backend, plus a second start against a different API base URL using the same image.
- **Estimated Effort**: S
- **Owner Role**: DevOps Engineer
- **Status**: TODO

#### Task ID: T-0.DEPLOY.03
- **Title**: Compose stack — database, backend and frontend as separate services
- **Description**: One compose file bringing up PostgreSQL, the backend and the frontend as separate containers, consuming each application's **published image** (with a development override for local builds), so neither application repository needs its sibling checked out: shared network, health checks, start ordering, persistent database storage, and configuration through environment variables.
- **Plan source**: §1 principle "one container per service", §3 row "Delivery & Packaging", §7.4
- **Scope**: The three named services, plus the worker service added by T-0.REPORT.01 when the first scheduled job exists. Not production orchestration — see Open Questions.
- **Variables / Config**: database credentials and volume, `api_base_url`, `cors_allowed_origins`, service ports.
- **Dependencies**: T-0.DEPLOY.01, T-0.DEPLOY.02, T-0.CORE.03
- **Acceptance Criteria**: One command starts the database, backend and frontend as three separate containers, with the database healthy before the backend starts; the images come from the registry, so the stack comes up with **neither application repository checked out** (separate repositories); the frontend reaches the backend over the compose network and the backend reaches the database, while the frontend has no route to the database (API-first, verified rather than assumed); database data survives a full stack restart; each service's health is observable.
- **Evidence**: The stack brought up from clean with the three containers listed, a create-then-restart data check, and the frontend-to-database isolation demonstrated.
- **Estimated Effort**: M
- **Owner Role**: DevOps Engineer
- **Status**: TODO

### Stream: `CICD`

#### Task ID: T-0.CICD.01
- **Title**: Backend repository pipeline
- **Description**: The backend repository's own pipeline: run the T-0.CORE.02 layers on every change, gate on the ledger-integrity check, and publish the backend image.
- **Plan source**: §7.5 (separate repositories, decided 2026-09-17)
- **Scope**: The backend repository's pipeline only; the frontend repository's pipeline is T-0.CICD.02. No environment-specific infrastructure beyond what the stack and packaging decisions require.
- **Variables / Config**: `container_registry` (where the image is published).
- **Dependencies**: T-0.STACK.01, T-0.CORE.02, T-0.DEPLOY.01
- **Acceptance Criteria**: A change runs unit, integration and ledger-integrity checks; a failing check blocks merge; the backend image publishes to the registry tagged with the commit, with the frontend repository absent; a deploy step exists and is reproducible.
- **Evidence**: A deliberately failing branch blocked by the pipeline, and a passing run that published the backend image.
- **Estimated Effort**: M
- **Owner Role**: DevOps Engineer
- **Status**: TODO

#### Task ID: T-0.CICD.02
- **Title**: Frontend repository pipeline
- **Description**: The frontend repository's own pipeline: typecheck and build against the published contract artifact, then publish the frontend image.
- **Plan source**: §7.5 (separate repositories, decided 2026-09-17)
- **Scope**: The frontend repository's pipeline only. It must not require the backend repository to be checked out or built.
- **Variables / Config**: `container_registry`; `api_base_url` must not be baked in at build time (see T-0.DEPLOY.02).
- **Dependencies**: T-0.CICD.01, T-0.API.02, T-0.DEPLOY.02
- **Acceptance Criteria**: A change in the frontend repository typechecks against the published contract and publishes its image with the backend repository absent; a contract change that breaks the client fails this pipeline instead of failing at runtime; the image is tagged with the frontend commit.
- **Evidence**: A passing run producing the frontend image from the frontend repository alone, plus a deliberately stale contract reference failing the typecheck.
- **Estimated Effort**: M
- **Owner Role**: DevOps Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-0.X.GATE
- **Title**: Phase 0 exit check — foundations ready for Phase 1
- **Description**: Verify the derived Phase 0 foundations before any Phase 1 work starts. The plan defines no Phase 0; these checks are derived from §3 and §7.
- **Plan source**: derived from §3 and §7 (no plan-stated exit criteria)
- **Scope**: Verification only — no fixing beyond noting what failed.
- **Variables / Config**: all Phase 0 variables exist as configuration, not literals.
- **Dependencies**: T-0.STACK.01, T-0.CICD.01, T-0.CICD.02, T-0.CORE.01, T-0.CORE.02, T-0.CORE.03, T-0.MODELS.01, T-0.API.01, T-0.API.02, T-0.AUDIT.01, T-0.AUDIT.02, T-0.PARTY.01, T-0.WF.01, T-0.SEC.01, T-0.INT.01, T-0.REPORT.01, T-0.LOC.01, T-0.DEPLOY.01, T-0.DEPLOY.02, T-0.DEPLOY.03
- **Acceptance Criteria**:
  - A written stack decision exists covering backend, frontend, database, queue, search
  - CI runs unit, integration and ledger-integrity checks and blocks on failure
  - The posting primitive rejects unbalanced entries and is the only posting entry point
  - Company scoping is enforced; a second company cannot see the first company's data
  - Posted ledger and stock entries cannot be updated or deleted; masters soft-delete only
  - Every master and transaction change is attributable in the audit trail
  - One party holds multiple roles without duplication
  - A configured approval chain runs a document end to end
  - Field-level permission denial is enforced at the API boundary
  - Duplicate inbound integration delivery is processed once; failed outbound sends are retried and logged
  - A scheduled report runs, is scoped by permission, and records failure
  - At least one complete localization pack imports cleanly into a test company
  - Phase 1 domain models and API contracts are complete for Account, Journal Entry/GL, Item, Item Variant, UOM Conversion, Warehouse/Location, Stock Ledger Entry
  - The API contract is published from the running backend and the frontend consumes it through a generated client, holding no direct database access (API-first)
  - One command brings up the database, backend and frontend as three separate containers, database healthy first, with its data surviving a stack restart
  - Both applications build and publish from their own repositories — no sibling checkout, no shared source tree — with the contract artifact as the only coupling (separate repositories)
- **Evidence**: The checks above, each with its output; plus the Open Questions answered (stack recorded, target markets confirmed, any threshold defaults decided).
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Phase 1 — Foundation (Core Accounting + Inventory)

### Stream: `ACCT` — Accounting & Financial Management (§2.1)

#### Task ID: T-1.ACCT.01
- **Title**: Chart of Accounts — hierarchical tree with localization templates
- **Description**: Model and expose the Account entity as a tree under the five classes Assets, Liabilities, Equity, Income, Expense, seedable from a localization pack's CoA template, with codes and hierarchy integrity.
- **Plan source**: §2.1 Core Components (CoA – tree hierarchy … with localization support), §4 Phase 1 bullet 1, §5 Account
- **Scope**: Account CRUD, hierarchy, classification, template import. No balances, no statements.
- **Variables / Config**: `company_id`, `coa_template` (T-0.LOC.01), `fiscal_year_start`.
- **Dependencies**: T-0.MODELS.01, T-0.CORE.03, T-0.LOC.01, T-0.AUDIT.02
- **Acceptance Criteria**: Accounts nest to arbitrary depth under the five classes; a child cannot mix classes with its parent; codes are unique per company; an account referenced by a posting cannot be reparented into another class; a CoA template imports without manual fix-up; a parent account with children cannot be soft-deleted while children remain active.
- **Evidence**: A tree built from a pack template, a rejected class-mixing case, and a rejected delete-with-children case.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: TODO

#### Task ID: T-1.ACCT.02
- **Title**: General Ledger — immutable journal entry storage
- **Description**: Persist journal entries (posting date, account, debit, credit, party links) as an immutable ledger built on the T-0.CORE.01 primitive, with drill-down to the originating document.
- **Plan source**: §2.1 Core Components (GL – immutable transaction ledger (posting date, account, debit, credit, party links)), §4 Phase 1 bullet 1, §5 Journal Entry / GL Transaction
- **Scope**: Storage, ledger read views and source linking. No statements or reports (T-1.ACCT.07), no period locking (T-1.ACCT.04).
- **Variables / Config**: `company_id`, `base_currency`, `transaction_currency` (recorded, conversion is T-1.ACCT.05), party link.
- **Dependencies**: T-0.CORE.01, T-0.AUDIT.01, T-1.ACCT.01
- **Acceptance Criteria**: Every entry stores posting date, account, debit, credit and party links where applicable; posted entries are append-only (no update, no delete); an entry records the document that produced it; the account balance is derivable from the entries alone.
- **Evidence**: A posted entry with its source link, plus rejected update/delete attempts.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.ACCT.03
- **Title**: Automatic double-entry posting interface for all sub-modules
- **Description**: Publish the one posting interface that inventory, procurement, sales, POS, manufacturing and payroll call, so no module writes journal lines of its own.
- **Plan source**: §2.1 Key Capabilities (Automatic double-entry posting from all sub-modules), §3 row "Double-Entry Integrity", §6 metric 1
- **Scope**: The posting interface and its account-mapping inputs. Not the per-module call sites (each owning phase wires its own).
- **Variables / Config**: `company_id`; account mapping supplied by the caller.
- **Dependencies**: T-1.ACCT.02, T-0.CORE.02
- **Acceptance Criteria**: Every sub-module posts through this interface (verified by the ledger-integrity check finding no other writer); an unbalanced or single-line call is rejected; postings are atomic with the calling transaction (a rolled-back document leaves no journal entry); the interface is documented for the later phases.
- **Evidence**: The ledgers-integrity check from T-0.CORE.02 green with Phase 1 writers connected, plus an atomicity check (failed document → no entry).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.ACCT.04
- **Title**: Period locking and posting audit trail
- **Description**: Allow periods to be closed so no further postings (or only controlled ones) land in them, with every lock/unlock and every posting recorded in the audit trail.
- **Plan source**: §2.1 Key Capabilities (Period locking and audit trail)
- **Scope**: Locking, unlocking authority and the posting trail. No year-end closing entries (not named in the plan).
- **Variables / Config**: `period_lock_granularity`, `rbac_roles` (who may lock/unlock), `fiscal_year_start`.
- **Dependencies**: T-1.ACCT.02, T-0.AUDIT.02, T-0.SEC.01
- **Acceptance Criteria**: A posting dated inside a locked period is rejected; unlocking requires a permission and is itself audited with actor and reason; locking does not alter existing entries.
- **Evidence**: A rejected back-dated posting and an audited unlock record.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.ACCT.05
- **Title**: Multi-currency engine and central FX rate service
- **Description**: Maintain the currency master, hold rates by date, sync daily rates from the configured source, and convert amounts at posting time using the central service.
- **Plan source**: §2.1 Core Components (Multi-Currency Engine – daily FX rate sync), §3 row "Multi-Currency", §4 Phase 1 bullet 2
- **Scope**: Currency master, rate storage, sync job, conversion API. Gain/loss calculation is T-1.ACCT.06.
- **Variables / Config**: `base_currency`, `transaction_currency`, `fx_rate_source`, `fx_sync_schedule`.
- **Dependencies**: T-1.ACCT.02, T-0.INT.01
- **Acceptance Criteria**: Rates are stored per currency pair per date and are never overwritten silently for a past date; the daily sync populates the current day and reports failure loudly; a document in a foreign currency stores its rate, its foreign amount and its base amount; conversion uses the rate for the posting date, not today's.
- **Evidence**: A foreign-currency posting whose base amount equals the dated rate applied to the foreign amount, plus a failed-sync report.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.ACCT.06
- **Title**: Realized and unrealized FX gain/loss
- **Description**: Calculate and post unrealized revaluation for open foreign-currency balances at period end, and realized gain/loss on settlement of foreign-currency documents.
- **Plan source**: §2.1 Core Components (unrealized/realized gain/loss), §3 row "Multi-Currency"
- **Scope**: The gain/loss calculations and their postings. No treasury or hedging features (not named).
- **Variables / Config**: `base_currency`, `fx_rate_source`, `period_lock_granularity`.
- **Dependencies**: T-1.ACCT.05, T-1.ACCT.03
- **Acceptance Criteria**: Settlement of a foreign-currency document posts the realized difference to the configured gain/loss account and balances; revaluation at a chosen date posts the unrealized difference and is reversible/repeatable without double-counting; the gains/losses reconcile to the change in rate applied to the open balance.
- **Evidence**: One realized and one unrealized case with the resulting entries, both balanced.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: TODO

#### Task ID: T-1.ACCT.07
- **Title**: Basic financial reports — Trial Balance, P&L, Balance Sheet, Cash Flow
- **Description**: Produce the four statements from the GL, scoped by company and period, delivered through the reporting framework.
- **Plan source**: §2.1 Key Capabilities (Financial statements (Trial Balance, P&L, Balance Sheet, Cash Flow)), §4 Phase 1 bullet 5 (Basic financial reports)
- **Scope**: The four statements at basic depth. Advanced analytics and the broader scheduled catalogue are Phase 6.
- **Variables / Config**: `company_id`, `fiscal_year_start`, `period_lock_granularity`, `base_currency`, `report_schedule`.
- **Dependencies**: T-1.ACCT.03, T-0.REPORT.01
- **Acceptance Criteria**: Trial Balance debits equal credits for any selected period; P&L and Balance Sheet foot to the GL and the Balance Sheet balances (assets = liabilities + equity); Cash Flow is derivable from the entries; each report is scoped by company and permission and can be scheduled.
- **Evidence**: The four reports for a period with a non-trivial dataset, plus the footing checks.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: TODO

### Stream: `INV` — Inventory & Warehouse Management (§2.2)

#### Task ID: T-1.INV.01
- **Title**: Item master — SKU, variants, UOM conversion, barcode/QR
- **Description**: Model and expose the Item, Item Variant and UOM Conversion entities, with barcode/QR identifiers, and a base UOM with conversion factors to alternate UOMs.
- **Plan source**: §2.2 Core Components (Item Master (SKU, variants, UOM conversion, barcode/QR)), §4 Phase 1 bullet 3, §5 Item, Item Variant, UOM Conversion
- **Scope**: Item identity, attributes, variants, UOM conversion, identifiers. Valuation attributes (costing method) belong to T-1.INV.04; batch/serial attributes to T-1.INV.08 and T-1.INV.09.
- **Variables / Config**: `uom_conversion_factor`, `item_variant_attributes`, `barcode_symbology`, `traceability_mode`, `company_id`.
- **Dependencies**: T-0.MODELS.01, T-0.AUDIT.02
- **Acceptance Criteria**: A variant matrix is defined by attributes without creating duplicate SKUs; a barcode is unique system-wide and scanning it resolves to exactly one item/variant; UOM conversion is lossless for a round trip within its declared precision; conversion factors of zero are rejected.
- **Evidence**: An item with two variants and a two-step UOM conversion (base → box → pallet) that round-trips exactly, plus a duplicate-barcode rejection.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.02
- **Title**: Multi-warehouse location hierarchy (Warehouse → Zone → Aisle → Bin)
- **Description**: Model the hierarchical location tree and let every stock entry reference a leaf location, with tree integrity and per-location capacity/status at basic depth.
- **Plan source**: §2.2 Core Components (Multi-Warehouse Hierarchy (Warehouse → Zone → Aisle → Bin)), §4 Phase 1 bullet 3 (basic Warehouse hierarchy), §1 principle 4
- **Scope**: The hierarchy and location referencing. No bin-level capacity planning or put-away strategies (not named in the plan).
- **Variables / Config**: `warehouse_hierarchy_levels`, `company_id`.
- **Dependencies**: T-0.MODELS.01
- **Acceptance Criteria**: The four levels nest with the named types; a stock entry cannot reference a non-leaf location; a location with stock cannot be removed; the hierarchy is retrievable as a tree per company.
- **Evidence**: A tree with stock posted at bin level and a rejected attempt to post at warehouse level.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.03
- **Title**: Stock ledger entry model (quantity + value, append-only)
- **Description**: Persist every stock movement as an immutable ledger entry carrying quantity and value, the item, the location, and the source document.
- **Plan source**: §2.2 Core Components (Stock Ledger & Valuation), §4 Phase 1 bullet 3, §5 Stock Ledger Entry (qty + value), §1 principle 5
- **Scope**: The entry model and its read views. Valuation maths is T-1.INV.04.
- **Variables / Config**: `company_id`, `warehouse_hierarchy_levels` (via location), `transaction_currency` (value currency).
- **Dependencies**: T-1.INV.01, T-1.INV.02, T-0.CORE.01, T-0.AUDIT.01
- **Acceptance Criteria**: Every movement writes one entry with quantity and value; entries are append-only; the current quantity and value of an item at a location equal the sum of its entries; a movement without a source document is rejected.
- **Evidence**: A sequence of movements whose summed ledger equals the reported on-hand quantity and value.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.04
- **Title**: Valuation engine — FIFO / Moving Average / Standard Cost with real-time valuation
- **Description**: Compute stock value per the configured costing method and expose real-time valuation for any item and location, including standard-cost revaluation handling.
- **Plan source**: §2.2 Core Components (Stock Ledger & Valuation (FIFO, Moving Average, Standard Cost)), §2.2 Key Capabilities (Real-time stock valuation), §4 Phase 1 bullet 3/4
- **Scope**: The costing methods, their selection and real-time valuation. Physical counts and adjustments are T-1.INV.06; GL posting is T-1.INV.07.
- **Variables / Config**: `costing_method`, `valuation_scope`.
- **Dependencies**: T-1.INV.03
- **Acceptance Criteria**: Each of the three methods produces the method-correct value for the same movement sequence; the method is selectable at the configured scope and changing it does not rewrite history; valuation is available without a batch job; a method that cannot value an item (e.g. standard cost with no standard set) fails loudly rather than valuing at zero.
- **Evidence**: The same movement sequence valued under all three methods with the expected figures, and a missing-standard-cost failure case.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: TODO

#### Task ID: T-1.INV.05
- **Title**: Stock transactions — Receipt, Issue, Transfer
- **Description**: Implement the three Phase 1 stock transactions, each writing stock ledger entries and honouring the location hierarchy and UOM conversion.
- **Plan source**: §2.2 Core Components (Stock Transactions (Receipt, Issue, Transfer, Reconciliation)), §4 Phase 1 bullet 4
- **Scope**: These three transaction types only. Reconciliation/adjustment is split out as T-1.INV.06 (the plan names them in §2.2 and schedules none, so they are placed with the transaction family in Phase 1 — see Open Questions).
- **Variables / Config**: `costing_method` (via valuation), `uom_conversion_factor`, `warehouse_hierarchy_levels`.
- **Dependencies**: T-1.INV.04
- **Acceptance Criteria**: Receipt increases quantity and value at the target location; issue decreases it and cannot drive quantity negative unless negative stock is explicitly permitted; transfer moves quantity and value between locations with no net change in total quantity/value; entries in a foreign UOM convert to the base UOM correctly; each transaction is atomic.
- **Evidence**: One receipt, one issue and one cross-location transfer with before/after totals, plus a rejected negative-stock issue.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.06
- **Title**: Physical count and reconciliation/adjustment workflow
- **Description**: Support counting a location (or a subset of items), recording counted quantities, and posting the resulting adjustment with approval and an audit trail.
- **Plan source**: §2.2 Core Components (Reconciliation), §2.2 Key Capabilities (Physical count and adjustment workflows), §1 principle 6
- **Scope**: Count sheets, variance calculation, adjustment posting. No cycle-count scheduling automation (not named in the plan).
- **Variables / Config**: `approval_levels`/`approval_thresholds` (adjustment approval), `costing_method` (adjustment value).
- **Dependencies**: T-1.INV.05, T-0.WF.01
- **Acceptance Criteria**: A count produces a variance per item against system quantity; posting an adjustment creates ledger entries for exactly the variance; an adjustment above the configured threshold requires approval before it affects the ledger; the count, the approver and the adjustment are all auditable.
- **Evidence**: A count with a positive and a negative variance, both posted and auditable, plus a blocked above-threshold adjustment awaiting approval.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.07
- **Title**: Post stock movements to the GL inventory account and reconcile valuation to GL
- **Description**: Wire stock ledger activity into the GL through T-1.ACCT.03 so the inventory account reflects stock value, and provide the reconciliation that proves the two agree (metric 2 of §6).
- **Plan source**: §2.2 Key Capabilities (Real-time stock valuation), §3 row "Double-Entry Integrity", §6 metric 2, §4 Phase 1 exit criteria
- **Scope**: The posting wiring and the reconciliation check. Not the valuation maths (T-1.INV.04) nor the statements (T-1.ACCT.07).
- **Variables / Config**: account mapping (inventory, stock adjustment, COGS), `costing_method`.
- **Dependencies**: T-1.INV.05, T-1.ACCT.03
- **Acceptance Criteria**: Every receipt, issue, transfer and adjustment posts a balanced entry to the chart of accounts; the GL inventory balance equals the sum of stock ledger values for the same period to the unit of currency precision; a deliberate mismatch is detected and reported, not silently accepted; reconciliation can be run per company and period.
- **Evidence**: The reconciliation report for a period showing zero difference, and a mismatch case that is reported.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.08
- **Title**: Batch/Lot tracking with expiry
- **Description**: Extend the stock ledger entry and the stock transactions with batch/lot identity, expiry dates and a FEFO issue suggestion, for items whose `traceability_mode` is Batch/Lot.
- **Plan source**: §2.2 Core Components (Traceability (Batch/Lot with expiry, Serial numbers)), §5 Batch / Serial — **scheduled in Phase 1 by the 2026-09-17 revision** (was §4 Phase 6)
- **Scope**: Batch identity across the Phase 1 stock transactions and the Phase 2 receipts that reuse them. Serial tracking is T-1.INV.09. Extends the entry model of T-1.INV.03 rather than replacing it.
- **Variables / Config**: `traceability_mode` (Batch/Lot), `batch_expiry_required`, `costing_method` (batch-level valuation).
- **Dependencies**: T-1.INV.05, T-1.INV.03
- **Acceptance Criteria**: Batch identity is required on every movement of a batch-tracked item and a movement without it is refused; items whose mode is `None` are unaffected (no regression to T-1.INV.05); batch quantities reconcile to the item's total quantity; an expired batch cannot be issued without an explicit, audited override; issue suggestion follows FEFO; valuation remains correct per batch under the item's costing method.
- **Evidence**: A batch-tracked item moved through receipt, issue and transfer with batch reconciliation, an expired-batch refusal, and an untracked item proven unaffected.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-1.INV.09
- **Title**: Serial number tracking
- **Description**: Track individually serialised units through receipt, issue and transfer, with a status per serial, for items whose `traceability_mode` is Serial. Sales and returns exercise it from Phase 3.
- **Plan source**: §2.2 Core Components (Traceability (… Serial numbers)), §5 Batch / Serial — **scheduled in Phase 1 by the 2026-09-17 revision** (was §4 Phase 6)
- **Scope**: Serial identity lifecycle and status within the Phase 1 stock transactions. Service/warranty features are not named in the plan.
- **Variables / Config**: `traceability_mode` (Serial), `allow_negative_stock` (never).
- **Dependencies**: T-1.INV.05, T-1.INV.03
- **Acceptance Criteria**: A serial-tracked item's quantity always equals its count of distinct serials in stock; a serial can only be in one place at one time; receiving, transferring and issuing a serial transitions its status correctly; a duplicate serial within an item is refused while the same value under a different item is allowed.
- **Evidence**: One serial taken through receipt → transfer → issue with its status history, plus a duplicate-serial refusal.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-1.X.GATE
- **Title**: Phase 1 exit check
- **Description**: Verify the plan's Phase 1 exit criteria against the running system.
- **Plan source**: §4 Phase 1 Exit Criteria (plan amended 2026-09-17): *Ability to post balanced entries and maintain accurate stock valuation.* · *Batch/lot and serial identity is enforced on every movement of a tracked item.*
- **Scope**: Verification only.
- **Variables / Config**: `costing_method` (Moving Average), `period_lock_granularity` (month), `allow_negative_stock` (never), `base_currency`.
- **Dependencies**: T-1.ACCT.01, T-1.ACCT.02, T-1.ACCT.03, T-1.ACCT.04, T-1.ACCT.05, T-1.ACCT.06, T-1.ACCT.07, T-1.INV.01, T-1.INV.02, T-1.INV.03, T-1.INV.04, T-1.INV.05, T-1.INV.06, T-1.INV.07, T-1.INV.08, T-1.INV.09
- **Acceptance Criteria**:
  - Balanced entries can be posted from more than one source module through a single posting interface, and unbalanced ones are refused — §6 metric 1
  - The ledger-integrity check reports zero imbalances over the whole dataset
  - Stock valuation is accurate under each configured costing method, including after receipts, issues, transfers and adjustments
  - Stock valuation matches the GL inventory account within currency precision — §6 metric 2
  - Period locking prevents back-dated postings into a closed period
  - A foreign-currency document posts with its dated rate, and realized/unrealized gain/loss posts balanced entries
  - Trial Balance balances and the Balance Sheet balances for a non-trivial dataset
  - Batch/lot and serial identity is enforced on every movement of a tracked item, and untracked items are unaffected (Phase 1 exit criterion added 2026-09-17)
- **Evidence**: Each check above with its output; the reconciliation report; the statements.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Phase 2 — Procurement & Payables

### Stream: `PROC` — Supply Chain & Procurement (§2.3)

#### Task ID: T-2.PROC.01
- **Title**: Supplier master
- **Description**: Build the supplier role on the shared Party master with the procurement-specific data: contacts, payment terms, bank details, tax identifiers and currency.
- **Plan source**: §2.3 Core Components, §4 Phase 2 bullet 4 (Supplier master), §5 Party
- **Scope**: Supplier master data only. Performance scoring is T-2.PROC.09.
- **Variables / Config**: `company_id`, `transaction_currency`, `tax_pack` (identifiers/fields).
- **Dependencies**: T-0.PARTY.01, T-0.AUDIT.02
- **Acceptance Criteria**: A supplier is created from an existing or new party with the supplier role; one party can be both supplier and customer without duplicate records; bank and tax details are validated at entry; suppliers soft-delete only and a supplier referenced by a document cannot be removed.
- **Evidence**: A dual-role party exercised as both supplier and customer, plus a blocked delete of a referenced supplier.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.02
- **Title**: Purchase requisitions with approval workflow
- **Description**: Let requesters raise purchase requisitions and route them through the configured multi-level approval chain before they can progress.
- **Plan source**: §2.3 Core Components (Purchase Requisitions + approval workflows), §4 Phase 2 bullet 1, §3 row "Workflows & Approvals"
- **Scope**: Requisition document, lines, and approval routing. Sourcing happens in RFQ (T-2.PROC.03).
- **Variables / Config**: `approval_levels`, `approval_thresholds`, `company_id`.
- **Dependencies**: T-2.PROC.01, T-0.WF.01
- **Acceptance Criteria**: A requisition above the configured threshold cannot be sourced until every level approves; rejection closes or returns it per the workflow; an approved requisition is immutable except by a new revision; approval history is complete and auditable.
- **Evidence**: A requisition driven through a two-level chain, plus a blocked progression before final approval.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.03
- **Title**: RFQ creation and supplier response capture
- **Description**: Raise a request for quotation against approved requisition lines, invite suppliers, and record their quoted prices, lead times and validity.
- **Plan source**: §2.3 Core Components (Request for Quotation (RFQ) & Supplier Portal)
- **Scope**: RFQ issue and response capture in the application. The supplier-facing portal is Phase 6 (T-6.PORTAL.01).
- **Variables / Config**: `tax_pack` (quote tax), `transaction_currency`, `approval_levels` (if an RFQ needs approval — not stated in the plan).
- **Dependencies**: T-2.PROC.02
- **Acceptance Criteria**: An RFQ can be issued to multiple suppliers with a response deadline; responses are recorded per line per supplier; a late response is recorded as late rather than silently accepted; the RFQ shows which requisition lines each response covers.
- **Evidence**: One RFQ issued to two suppliers with both responses captured, including one after the deadline.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.04
- **Title**: Comparative statement matrix
- **Description**: Present supplier responses side by side per line with price, tax, lead time and landed comparison, so an award can be justified.
- **Plan source**: §2.3 Key Capabilities (Comparative statement matrix)
- **Scope**: Comparison view and its export. Awarding is T-2.PROC.05.
- **Variables / Config**: `tax_pack`, `transaction_currency`.
- **Dependencies**: T-2.PROC.03
- **Acceptance Criteria**: Every responding supplier appears per line with its quoted values; the comparison uses a consistent basis (converted to base currency and tax-inclusive as configured) and labels the basis; a supplier that did not respond is distinguishable from one that quoted zero; the matrix can be exported.
- **Evidence**: A matrix for an RFQ with two responses and one non-response, exported, with the comparison basis stated.
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.05
- **Title**: Automated PO generation from RFQ award
- **Description**: Award one or more RFQ lines to suppliers and generate the purchase order(s) automatically from the winning responses.
- **Plan source**: §2.3 Key Capabilities (Automated PO generation from RFQ)
- **Scope**: Award and PO generation. The PO's own lifecycle is T-2.PROC.06.
- **Variables / Config**: `transaction_currency`, `tax_pack`, `approval_levels` (if PO approval applies).
- **Dependencies**: T-2.PROC.03, T-2.PROC.04
- **Acceptance Criteria**: Awarding generates a PO whose lines carry the awarded price, supplier, requisition reference and required date with no re-keying; an award cannot exceed the requisitioned quantity without explicit override; a partial award leaves the remaining lines available for award.
- **Evidence**: One requisition awarded to two suppliers producing two POs, with the remaining quantity still awardable.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.06
- **Title**: Purchase order lifecycle — approve, amend, close
- **Description**: Manage the PO through approval, amendment with revision history, partial fulfilment and closure.
- **Plan source**: §2.3 Core Components (Purchase Orders), §4 Phase 2 bullet 1
- **Scope**: PO states and transitions. Receiving is T-2.PROC.07.
- **Variables / Config**: `approval_levels`, `approval_thresholds`.
- **Dependencies**: T-2.PROC.02, T-2.PROC.05
- **Acceptance Criteria**: A PO above threshold requires approval before it can be sent or received against; an amendment after approval produces a new revision without erasing the approved one; a PO cannot close with open receipts; the status transitions are recorded with actor and timestamp.
- **Evidence**: One PO approved, amended and closed, with the revision history and status trail.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.07
- **Title**: Goods Receipt Note (GRN) against a purchase order
- **Description**: Receive goods against PO lines into a warehouse location, creating stock ledger entries and recording rejected/partial receipts.
- **Plan source**: §2.3 Core Components (Goods Receipt Notes (GRN)), §4 Phase 2 bullet 2
- **Scope**: Receipt document and its stock effect. Invoice matching is T-2.MATCH.01.
- **Variables / Config**: `warehouse_hierarchy_levels`, `uom_conversion_factor`, `costing_method` (receipt value).
- **Dependencies**: T-2.PROC.06, T-1.INV.05
- **Acceptance Criteria**: A GRN increases stock at the chosen location and links to its PO lines; over-receipt beyond the PO quantity (plus any tolerance) is refused or explicitly overridden; a partial receipt leaves the remaining quantity receivable; the receipt's value flows into stock valuation.
- **Evidence**: One full and one partial receipt against the same PO, with the stock ledger and PO remaining quantities shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.08
- **Title**: Supplier tax compliance and tax rule application
- **Description**: Apply the localization pack's tax rules to procurement documents and validate supplier tax identifiers, producing tax-inclusive values that downstream AP postings reuse.
- **Plan source**: §2.3 Key Capabilities (tax compliance), §3 row "Localization"
- **Scope**: Tax rule application on procurement documents and supplier tax data. Statutory filing outputs are not named in the plan and are not built here.
- **Variables / Config**: `tax_pack`, `company_id`.
- **Dependencies**: T-2.PROC.01, T-0.LOC.01
- **Acceptance Criteria**: Tax is computed per the active pack for each document type; a supplier with an invalid or missing tax identifier is flagged before the document is approved; tax amounts on the PO, GRN and invoice share one basis so the later match compares like with like; no market is hard-coded.
- **Evidence**: One document set under a pack with a non-trivial rate set, and a flagged invalid supplier identifier.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Accounting) + Backend Engineer
- **Status**: TODO

#### Task ID: T-2.PROC.09
- **Title**: Supplier performance scoring and vendor scorecards
- **Description**: Score suppliers from actuals — on-time delivery, receipt-to-PO quantity variance, quality/rejection rate and price variance — and present a scorecard.
- **Plan source**: §2.3 Key Capabilities (Supplier performance scoring, Vendor scorecards), §4 Phase 2 bullet 4 (performance basics)
- **Scope**: Basic scoring from the data produced in Phase 2. Cross-period trend analytics belong to Phase 6.
- **Variables / Config**: scoring weights (per company — not stated in the plan), `company_id`.
- **Dependencies**: T-2.PROC.06, T-2.PROC.07
- **Acceptance Criteria**: Each score derives from recorded documents, not manual entry; the inputs (promised date vs receipt date, ordered vs received quantity, rejections) are traceable to their source documents; the scorecard shows the weights used; a supplier with no receipts is shown as unrated rather than scored zero.
- **Evidence**: Scorecards for two suppliers with different performance, each input traced to a document.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `AP` — Accounts Payable (§2.1)

#### Task ID: T-2.AP.01
- **Title**: Supplier invoice entry and GL posting
- **Description**: Record supplier invoices (with tax and currency) against suppliers and, where applicable, PO/GRN references, and post them to the GL through the shared posting interface.
- **Plan source**: §2.1 Core Components (Accounts Payable (AP) – supplier invoices), §4 Phase 2 bullet 3, §3 row "Double-Entry Integrity"
- **Scope**: Invoice capture and its posting. Matching is T-2.MATCH.01; settlement is T-2.AP.04.
- **Variables / Config**: `transaction_currency`, `tax_pack`, account mapping (payables control, expense).
- **Dependencies**: T-2.PROC.01, T-1.ACCT.03
- **Acceptance Criteria**: An invoice posts a balanced entry to the payables control account on the stated date; a duplicate supplier invoice (same supplier, number, amount) is refused; a foreign-currency invoice records rate, foreign and base amounts; the invoice is attributable to the document that produced it.
- **Evidence**: A posted invoice with its entry, plus a duplicate-invoice rejection.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.AP.02
- **Title**: AP aging report
- **Description**: Age open supplier balances by due date and bucketed periods, per supplier and per company.
- **Plan source**: §2.1 Core Components (AP – aging)
- **Scope**: Aging report and its export. Dunning for payables is not named in the plan (dunning belongs to AR).
- **Variables / Config**: `company_id`, `report_schedule`, aging buckets (per company — not stated).
- **Dependencies**: T-2.AP.01
- **Acceptance Criteria**: Aging buckets are configurable and stated on the report; each aged amount traces to an open invoice; the report total equals the AP control account balance; partial settlements reduce the correct bucket.
- **Evidence**: An aging report over a dataset with partial payments, reconciled to the control account.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.AP.03
- **Title**: Debit notes for supplier returns and adjustments
- **Description**: Issue debit notes against suppliers (returns, price adjustments) with their stock and GL effect.
- **Plan source**: §2.1 Core Components (AP – debit notes)
- **Scope**: Debit note document, stock return where applicable, and posting. Credit notes on the sales side are Phase 3 scope.
- **Variables / Config**: `tax_pack`, `transaction_currency`, `warehouse_hierarchy_levels` (return location).
- **Dependencies**: T-2.AP.01, T-1.INV.05
- **Acceptance Criteria**: A debit note posts the reverse of the corresponding invoice lines and balances; a return reduces stock at the correct location and value; a debit note cannot be raised for more than the invoice's remaining open amount without an explicit override that is recorded.
- **Evidence**: One debit note for a return and one for a price adjustment, both posted and reconciled against the source invoice.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.AP.04
- **Title**: Payment batches and payment run with settlement posting
- **Description**: Select open invoices into a payment batch, approve it, execute the payment run, and post the settlement — supporting partial and full settlement.
- **Plan source**: §2.1 Core Components (AP – payment batches), §4 Phase 2 bullet 3
- **Scope**: Batch selection, approval, execution and posting. Bank file generation for payables uses `bank_file_format` from the pack; automated bank transmission is not named in the plan.
- **Variables / Config**: `payment_batch_schedule`, `approval_levels`/`approval_thresholds`, `bank_file_format`, `transaction_currency`.
- **Dependencies**: T-2.AP.01, T-2.MATCH.01
- **Acceptance Criteria**: A batch cannot include an invoice that has failed 3-way matching (metric-facing, see T-2.MATCH.02) or is otherwise on hold; execution posts one balanced entry per settled invoice; partial settlement leaves the correct remaining balance; a batch cannot be executed twice; removing an invoice after execution is impossible.
- **Evidence**: One executed batch with a partial and a full settlement, plus a rejected attempt to include a held invoice.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-2.AP.05
- **Title**: AP to GL reconciliation and payables control check
- **Description**: Prove that open supplier balances equal the payables control account, per company and period, and report any difference.
- **Plan source**: §3 row "Double-Entry Integrity", §2.1 Key Capabilities (Automatic double-entry posting from all sub-modules)
- **Scope**: The reconciliation only. No new posting logic.
- **Variables / Config**: `company_id`, `base_currency`.
- **Dependencies**: T-2.AP.01, T-2.AP.04
- **Acceptance Criteria**: Subledger open balance equals the control account to currency precision for any period; an injected mismatch is reported rather than swallowed; the reconciliation runs per company.
- **Evidence**: A clean reconciliation plus a reported mismatch case.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `MATCH` — 3-way matching (§2.3)

#### Task ID: T-2.MATCH.01
- **Title**: 3-way match engine (PO ↔ GRN ↔ Supplier Invoice)
- **Description**: Compare purchase order, goods receipt and supplier invoice line by line with configurable tolerances, and mark each invoice as matched, partially matched or failed. This owns §6 metric 3 (> 95 % match rate).
- **Plan source**: §2.3 Core Components (3-way matching (PO ↔ GRN ↔ Supplier Invoice)), §2.3 Key Capabilities, §4 Phase 2 bullet 2 and exit criteria, §6 metric 3
- **Scope**: The comparison and its verdicts plus match-rate measurement. Held-invoice handling is T-2.MATCH.02; payment selection is T-2.AP.04.
- **Variables / Config**: `three_way_match_tolerance`, `three_way_match_target` (> 95 %), `tax_pack` (comparison basis), `transaction_currency`.
- **Dependencies**: T-2.PROC.07, T-2.AP.01
- **Acceptance Criteria**: Quantity, price and tax are compared per line within configured tolerances and each is reported separately when exceeded; the verdict is explainable (which line, which value, which tolerance); the match rate is measured over a rolling period and reported against the > 95 % target; a tolerance cannot be set so loose that a wrong-price invoice passes silently.
- **Evidence**: A matched invoice, a quantity-mismatch, a price-mismatch and a within-tolerance case, plus the measured match rate.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: TODO

#### Task ID: T-2.MATCH.02
- **Title**: Match exception and hold/release workflow
- **Description**: Hold invoices that fail matching, route exceptions for resolution, and allow release only through an authorised, audited override.
- **Plan source**: §2.3 Core Components (3-way matching), §3 row "Workflows & Approvals", §1 principle 6
- **Scope**: Hold/release and exception resolution. The tolerance maths is T-2.MATCH.01.
- **Variables / Config**: `approval_levels`, `approval_thresholds`, `rbac_roles` (who may override).
- **Dependencies**: T-2.MATCH.01, T-0.WF.01
- **Acceptance Criteria**: A failed invoice cannot enter a payment batch while held; release requires the configured permission and records actor, reason and the exception it resolves; an override does not alter the underlying match verdict (the exception stays visible for reporting); the match-rate metric counts overrides separately from clean matches.
- **Evidence**: One held invoice released with an audited reason, and one still-blocked invoice in a payment batch attempt.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-2.X.GATE
- **Title**: Phase 2 exit check
- **Description**: Verify the plan's Phase 2 exit criteria against the running system.
- **Plan source**: §4 Phase 2 Exit Criteria (verbatim): *End-to-end purchase-to-pay cycle with 3-way match.*
- **Scope**: Verification only.
- **Variables / Config**: `three_way_match_tolerance`, `three_way_match_target`, `approval_levels`.
- **Dependencies**: T-2.PROC.01 … T-2.PROC.09, T-2.AP.01 … T-2.AP.05, T-2.MATCH.01, T-2.MATCH.02
- **Acceptance Criteria**:
  - A requisition runs through approval → RFQ → comparative statement → automated PO → GRN → supplier invoice → 3-way match → payment batch → settlement with no manual re-keying
  - The 3-way match rate on the exercised dataset is measured and the engine reports it against the > 95 % target — §6 metric 3
  - Every step's postings balance, and the AP subledger equals the payables control account — §6 metrics 1 and 2 principles
  - A failed match blocks payment until an authorised, audited release
  - Supplier tax is applied per the localization pack on every document in the cycle
- **Evidence**: The end-to-end run with each document id, the measured match rate, the reconciliations, and the blocked-then-released exception.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Phase 3 — Sales, Receivables & POS

### Stream: `SALES` — Sales & CRM (§2.5)

#### Task ID: T-3.SALES.01
- **Title**: Customer master with credit limits
- **Description**: Build the customer role on the shared Party master, including contacts, billing/shipping addresses, payment terms, currency and the credit limit field.
- **Plan source**: §2.5 Core Components, §4 Phase 3 bullet 1 (Customer master, Credit limits), §5 Party
- **Scope**: Customer master data and the stored limit. Enforcement is T-3.AR.06 and the order-time check is part of T-3.SALES.04.
- **Variables / Config**: `credit_limit`, `credit_check_mode`, `transaction_currency`, `company_id`.
- **Dependencies**: T-0.PARTY.01, T-0.AUDIT.02
- **Acceptance Criteria**: A customer is created from an existing or new party with the customer role and can also be a supplier without duplication; a credit limit is stored (including zero meaning "no credit") and distinguishes "no limit set" from "limit zero"; addresses are reusable across documents; referenced customers cannot be deleted.
- **Evidence**: A dual-role party as customer and supplier, plus the two credit-limit states distinguished.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.SALES.02
- **Title**: Lead and opportunity pipeline (Kanban)
- **Description**: Track leads and opportunities, move them through configurable pipeline stages on a Kanban board, and record value, owner and expected close date.
- **Plan source**: §2.5 Core Components (Lead & Opportunity pipeline (Kanban))
- **Scope**: Pipeline stages, movement and board. No marketing automation or campaign attribution (not named in the plan).
- **Variables / Config**: pipeline stages (per company — plan names no stages), `rbac_roles`, `company_id`.
- **Dependencies**: T-3.SALES.01, T-0.SEC.01
- **Acceptance Criteria**: Stages are configurable without code change; moving a card records who moved it and when; a won opportunity can be converted to a quotation with the customer details carried over (note: the pipeline slot in §4 Phase 3 is inferred — see Open Questions); a lost opportunity records a reason; the board respects field-level permissions so values can be hidden from some roles.
- **Evidence**: One opportunity moved from first to won stage and converted, with the stage history and a permission-restricted value.
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: TODO

#### Task ID: T-3.SALES.03
- **Title**: Quotations and quotation-to-order conversion
- **Description**: Create quotations with priced lines and validity, and convert an accepted quotation into a sales order without re-keying.
- **Plan source**: §2.5 Key Capabilities (Quotation → Order conversion), §4 Phase 3 bullet 2
- **Scope**: Quotation document and the conversion path. Pricing comes from T-3.SALES.06/07.
- **Variables / Config**: `transaction_currency`, `tax_pack`, quotation validity (plan states none), `pricing_rule_dimensions`.
- **Dependencies**: T-3.SALES.02
- **Acceptance Criteria**: A quotation prices its lines from the pricing engine at the time of quote and records which rule applied; an expired quotation cannot be converted without re-pricing; conversion produces an order with identical lines and links back to the quotation; a quotation cannot be converted twice.
- **Evidence**: One quotation priced by rule, converted once, with a second conversion refused.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.SALES.04
- **Title**: Sales orders with credit checking
- **Description**: Manage sales orders through confirmation, and apply the customer's credit check at order time according to the configured mode.
- **Plan source**: §2.5 Core Components (Sales Orders & Fulfillment (credit checks, pick lists, shipping)), §4 Phase 3 bullet 1/2
- **Scope**: Order document and the order-time credit decision. Exposure calculation across open AR is T-3.AR.06; fulfilment is T-3.SALES.05.
- **Variables / Config**: `credit_limit`, `credit_check_mode` (`off`/`warn`/`block`), `transaction_currency`.
- **Dependencies**: T-3.SALES.01, T-3.SALES.03
- **Acceptance Criteria**: In `block` mode an order that breaches the limit cannot be confirmed; in `warn` mode it can be confirmed only with a recorded acknowledgement; the decision shows the limit, the current exposure and the order value that produced it; changing `credit_check_mode` does not alter historical decisions.
- **Evidence**: The same order under `block` (refused), `warn` (acknowledged) and `off` (passes), with the exposure figures shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.SALES.05
- **Title**: Fulfilment — pick lists, shipping and stock issue
- **Description**: Generate pick lists from confirmed orders, record picked quantities and ship, issuing stock and posting the cost side of the sale through the shared posting interface.
- **Plan source**: §2.5 Core Components (Sales Orders & Fulfillment … pick lists, shipping), §4 Phase 3 bullet 2
- **Scope**: Fulfilment execution and its stock/GL effect. Invoicing is T-3.AR.01.
- **Variables / Config**: `warehouse_hierarchy_levels`, `uom_conversion_factor`, `costing_method` (cost of goods issued).
- **Dependencies**: T-3.SALES.04, T-1.INV.05, T-1.ACCT.03
- **Acceptance Criteria**: A pick list covers exactly the confirmed order lines; partial picking and partial shipping leave correct remaining quantities on the order; shipping issues stock at the correct warehouse with the correct valuation; a shipped order cannot be shipped again for the same quantity; every posting balances.
- **Evidence**: One order fulfilled in two shipments with the stock ledger, order remainders and postings shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.SALES.06
- **Title**: Pricing engine — customer tiers and volume rules
- **Description**: Resolve the applicable price for an item/customer/quantity using ordered rules over customer tier and volume, with an explainable outcome.
- **Plan source**: §2.5 Core Components (Pricing Rules & Discounts (customer tiers, volume…)), §4 Phase 3 bullet 4 (Pricing engine)
- **Scope**: The engine and tier/volume dimensions. Campaigns and coupons are T-3.SALES.07.
- **Variables / Config**: `pricing_rule_dimensions` (customer tier, volume), `discount_type`, `transaction_currency`.
- **Dependencies**: T-3.SALES.04
- **Acceptance Criteria**: Overlapping rules resolve deterministically, and the resolution order is stated on the priced line; a quote records the rule that applied so it can be reproduced later; a tier change does not retroactively change already-priced documents; the engine is used by both quotations and orders (one implementation, not two).
- **Evidence**: Overlapping rules on the same item resolved deterministically, with the winning rule shown.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.SALES.07
- **Title**: Campaigns and coupons
- **Description**: Add campaign-scoped and coupon-code discounts on top of the pricing engine, with validity windows and usage limits.
- **Plan source**: §2.5 Core Components (Pricing Rules & Discounts (… campaigns, coupons))
- **Scope**: Campaign and coupon dimensions and redemption. No marketing automation, no customer segmentation beyond tiers (not named).
- **Variables / Config**: `pricing_rule_dimensions` (campaign, coupon), `discount_type`, usage limits and validity (not stated in the plan).
- **Dependencies**: T-3.SALES.06
- **Acceptance Criteria**: A coupon applies only within its validity window and only up to its usage limit; a coupon cannot be stacked beyond the configured allowance; an expired or unknown coupon is rejected with a clear reason; redemption is recorded against the document that used it.
- **Evidence**: One coupon redeemed to its limit and then refused, plus an out-of-window rejection.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `AR` — Accounts Receivable (§2.1)

#### Task ID: T-3.AR.01
- **Title**: Customer invoicing with GL posting
- **Description**: Raise customer invoices from sales orders or standalone, with tax, currency and due dates, and post them to the GL through the shared posting interface.
- **Plan source**: §2.1 Core Components (Accounts Receivable (AR) – customer invoices), §4 Phase 3 bullet 3
- **Scope**: Invoice document and its posting. Aging is T-3.AR.02; settlement via gateway is T-3.AR.05.
- **Variables / Config**: `tax_pack`, `transaction_currency`, account mapping (receivables control, revenue).
- **Dependencies**: T-3.SALES.01, T-3.SALES.05, T-1.ACCT.03
- **Acceptance Criteria**: An invoice posts a balanced entry to the receivables control account; an invoice for goods already shipped references the shipment and does not re-issue stock; a duplicate invoice for the same order and amount is refused; tax is applied per the active pack.
- **Evidence**: A posted invoice with its entry, linked to its shipment, plus a duplicate rejection.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.AR.02
- **Title**: AR aging report
- **Description**: Age open customer balances by due date and configurable buckets, per customer and per company.
- **Plan source**: §2.1 Core Components (AR – aging)
- **Scope**: Aging report and export.
- **Variables / Config**: `company_id`, `report_schedule`, aging buckets (not stated).
- **Dependencies**: T-3.AR.01
- **Acceptance Criteria**: Each aged amount traces to an open invoice; the total equals the receivables control account; partial receipts reduce the correct bucket; buckets are configurable and shown on the report.
- **Evidence**: An aging report over a dataset with partial receipts, reconciled to the control account.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.AR.03
- **Title**: Recurring billing
- **Description**: Define recurring invoice templates and schedules, and generate invoices on their cadence.
- **Plan source**: §2.1 Core Components (AR – recurring billing)
- **Scope**: Templates, schedules and generation. Collection of the resulting invoices goes through the normal AR path.
- **Variables / Config**: `recurring_billing_cycle`, `tax_pack`, `transaction_currency`.
- **Dependencies**: T-3.AR.01
- **Acceptance Criteria**: A template generates invoices exactly on its cadence with no duplicates if the job reruns; a paused template generates nothing; a generated invoice is indistinguishable from a manual one for aging and dunning purposes; the generation is idempotent and its failures are reported.
- **Evidence**: A monthly template run twice for the same period producing one invoice, plus a paused template producing none.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.AR.04
- **Title**: Dunning — reminder levels, escalation and delivery
- **Description**: Define dunning levels by days past due with their templates, generate reminders for overdue invoices, and deliver them through the integration boundary by email/SMS.
- **Plan source**: §2.1 Core Components (AR – dunning), §4 Phase 3 bullet 3, §3 row "Integrations"
- **Scope**: Dunning levels, generation and delivery. Interest/penalty charging is not named in the plan and is not built here.
- **Variables / Config**: `dunning_levels`, `dunning_channel`, `report_schedule` cadence for reminder runs.
- **Dependencies**: T-3.AR.02, T-0.INT.01
- **Acceptance Criteria**: Each overdue invoice lands in exactly one level per run according to its days past due; an escalated invoice does not also receive the earlier level's reminder; delivery attempts and failures are visible in the delivery log; a settled invoice receives no further reminders; running the dunning job twice for a period does not duplicate reminders.
- **Evidence**: One invoice escalated through two levels and one settled mid-way, with the delivery log and the duplicate-run check.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.AR.05
- **Title**: Payment gateway webhooks and settlement posting
- **Description**: Receive payment-gateway webhooks, match them to open invoices, and post the settlement including any gateway fee, handling partial and failed payments.
- **Plan source**: §2.1 Core Components (AR – payment gateway webhooks), §4 Phase 3 bullet 3, §3 row "Integrations"
- **Scope**: Inbound payment events and their posting. Gateway-specific configuration uses the T-0.INT.01 boundary; refund/reversal of a captured payment is included only if the gateway event exists.
- **Variables / Config**: `integration_endpoints`, `transaction_currency`, `company_id`.
- **Dependencies**: T-3.AR.01, T-0.INT.01
- **Acceptance Criteria**: A duplicated webhook settles the invoice once (idempotency); an unmatched payment is parked and reported rather than silently dropped; a partial payment leaves the correct open amount; a failed payment leaves the invoice open and is recorded; the settlement posts a balanced entry.
- **Evidence**: A duplicated webhook, a partial payment and an unmatched payment, each with the resulting invoice state and posting.
- **Estimated Effort**: L
- **Owner Role**: Integration Engineer
- **Status**: TODO

#### Task ID: T-3.AR.06
- **Title**: Credit limit enforcement and exposure calculation
- **Description**: Compute a customer's live exposure across open invoices, unshipped orders and payments, and enforce the limit on new commitments per the configured mode.
- **Plan source**: §2.1 Core Components (AR – credit limits), §4 Phase 3 bullet 1
- **Scope**: Exposure and enforcement. The order-time check lives in T-3.SALES.04 and must reuse this calculation.
- **Variables / Config**: `credit_limit`, `credit_check_mode`, `transaction_currency`.
- **Dependencies**: T-3.SALES.01, T-3.AR.01
- **Acceptance Criteria**: Exposure includes open invoices and on-account receipts correctly, and the components are itemised; the same exposure figure is used by the order-time check (one implementation); an increase in exposure from a new invoice is immediately visible to the next order check; a limit change is audited.
- **Evidence**: An exposure statement itemised to its documents, and a subsequent order blocked on the updated figure.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.AR.07
- **Title**: AR to GL reconciliation and receivables control check
- **Description**: Prove that open customer balances equal the receivables control account per company and period, and report differences.
- **Plan source**: §3 row "Double-Entry Integrity"
- **Scope**: The reconciliation only.
- **Variables / Config**: `company_id`, `base_currency`.
- **Dependencies**: T-3.AR.01, T-3.AR.05
- **Acceptance Criteria**: Subledger open balance equals the control account to currency precision; an injected mismatch is reported; gateway settlements and partial receipts are reflected correctly.
- **Evidence**: A clean reconciliation plus a reported mismatch case.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `POS` — Point of Sale (§2.5)

#### Task ID: T-3.POS.01
- **Title**: POS sale transaction with barcode scanning and receipt (online)
- **Description**: Ring up a sale by scanning item barcodes, apply the pricing engine, take payment, produce a receipt, and post the sale and its stock movement.
- **Plan source**: §2.5 Core Components (POS (offline-capable, barcode, cash drawer, Z-Reports)), §4 Phase 3 bullet 5 (POS module (online first, offline later)), §6 metric 6
- **Scope**: **Online only** — offline operation is Phase 6 (T-6.OFFLINE.01). Runs in the frontend container against the API, with no direct database access (T-0.API.02). No loyalty or customer-display features (not named).
- **Variables / Config**: `pos_mode` (`online`), `barcode_symbology`, `pricing_rule_dimensions`, `payment_tender_types`, `pos_latency_budget`.
- **Dependencies**: T-1.INV.01, T-3.SALES.06, T-1.ACCT.03
- **Acceptance Criteria**: Scanning a barcode resolves to the item/variant and adds the correct line at the engine-resolved price; the sale posts a balanced entry and issues stock from the POS location; the receipt is reproducible from the stored sale; an unknown barcode is rejected rather than sold at zero; a sale cannot be completed without a settled tender.
- **Evidence**: A multi-line sale with its stock ledger entries, posting and reproducable receipt, plus an unknown-barcode rejection.
- **Estimated Effort**: L
- **Owner Role**: Full-stack Engineer
- **Status**: TODO

#### Task ID: T-3.POS.02
- **Title**: Cash drawer and payment tendering (cash, card, gateway, split)
- **Description**: Support the tenders a POS till accepts, including split payments across tenders, change calculation and drawer movements (paid in/out).
- **Plan source**: §2.5 Core Components (POS (… cash drawer …))
- **Scope**: Tendering and drawer movements. Shift open/close and cash count are T-3.POS.03.
- **Variables / Config**: `payment_tender_types`, `transaction_currency`.
- **Dependencies**: T-3.POS.01
- **Acceptance Criteria**: A sale settles only when the tendered total covers it; change is computed correctly and cannot go negative; a split payment records each tender separately with its own settlement path; drawer movements outside a sale are recorded with a reason and appear in the shift totals.
- **Evidence**: One cash sale with change, one split cash/card sale, and one paid-out, all reflected in shift totals.
- **Estimated Effort**: M
- **Owner Role**: Full-stack Engineer
- **Status**: TODO

#### Task ID: T-3.POS.03
- **Title**: Retail shift management
- **Description**: Open and close a till shift per store/terminal, with opening float, closing count, expected-versus-counted variance and the shift's document list.
- **Plan source**: §2.5 Key Capabilities (Retail shift management)
- **Scope**: Shift lifecycle and reconciliation of a shift. Z-Reports (the printed/exported close summary) are T-3.POS.04.
- **Variables / Config**: `cash_drawer_required`, `shift_definitions`, `payment_tender_types`.
- **Dependencies**: T-3.POS.02
- **Acceptance Criteria**: A shift cannot be opened twice on one terminal; sales cannot be rung up outside an open shift; closing records expected versus counted cash with the variance, and requires a reason when the variance is non-zero; a closed shift cannot accept new sales.
- **Evidence**: One shift opened, traded and closed, with a non-zero variance and its reason.
- **Estimated Effort**: M
- **Owner Role**: Full-stack Engineer
- **Status**: TODO

#### Task ID: T-3.POS.04
- **Title**: Z-Reports (shift and day close)
- **Description**: Produce the shift/day close report: sales by tender, tax, discounts, voids, drawer variance and totals, per terminal, shift and day.
- **Plan source**: §2.5 Core Components (POS (… Z-Reports))
- **Scope**: The close reports and their export. The reporting framework delivery mechanism is T-0.REPORT.01.
- **Variables / Config**: `report_schedule`, `company_id`, `payment_tender_types`.
- **Dependencies**: T-3.POS.03
- **Acceptance Criteria**: The Z-Report totals tie to the shift's constituent sales with no rounding gap; tax is broken out per the active pack; voids and refunds appear separately; the day report aggregates the shift reports exactly (equal to the sum, not approximately); the report is immutable once the shift is closed.
- **Evidence**: A shift Z-Report reconciled to its sales and a day report equal to the sum of its shifts.
- **Estimated Effort**: M
- **Owner Role**: Full-stack Engineer
- **Status**: TODO

#### Task ID: T-3.POS.05
- **Title**: POS to GL and stock reconciliation
- **Description**: Reconcile POS sales, tenders and stock movements against the GL and the stock ledger, exposing any difference.
- **Plan source**: §3 row "Double-Entry Integrity", §2.2 Real-time stock valuation principle
- **Scope**: The reconciliation only.
- **Variables / Config**: `company_id`, `report_schedule`.
- **Dependencies**: T-3.POS.04
- **Acceptance Criteria**: POS daily totals equal the corresponding GL revenue, tax and tender postings; POS stock movements reconcile to the stock ledger; a difference is reported per day and terminal rather than aggregated away; the reconciliation is re-runnable.
- **Evidence**: A clean reconciliation for a traded day plus a reported injected difference.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-3.POS.06
- **Title**: POS online transaction latency verification (< 2 s)
- **Description**: Measure and verify online POS transaction latency against §6's < 2 s target, and record the measured figures that the Phase 3 gate depends on.
- **Plan source**: §6 metric 6, §4 Phase 3 exit criteria
- **Scope**: Measurement and verification only. Fixing a failure is a new finding for the owner of the offending task; no optimisation work is bundled here.
- **Variables / Config**: `pos_latency_budget` (< 2 s), `pos_mode` (`online`).
- **Dependencies**: T-3.POS.05
- **Acceptance Criteria**: Latency is measured on a realistic dataset for a complete sale (scan → price → tender → post → receipt); the measured distribution is reported (not a single hand-picked run) and compared to the < 2 s budget; the measurement is repeatable in CI or a scheduled run.
- **Evidence**: The latency report with the measurement method, dataset size and the comparison against 2 s.
- **Estimated Effort**: S
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-3.X.GATE
- **Title**: Phase 3 exit check
- **Description**: Verify the plan's Phase 3 exit criteria against the running system.
- **Plan source**: §4 Phase 3 Exit Criteria (verbatim): *Complete order-to-cash cycle including POS sales.*
- **Scope**: Verification only.
- **Variables / Config**: `credit_check_mode`, `pos_latency_budget`, `dunning_levels`.
- **Dependencies**: T-3.SALES.01 … T-3.SALES.07, T-3.AR.01 … T-3.AR.07, T-3.POS.01 … T-3.POS.06
- **Acceptance Criteria**:
  - A lead runs through opportunity → quotation → order (credit-checked) → fulfilment → invoice → dunning → gateway settlement with no manual re-keying
  - A POS sale completes end to end, including shift and Z-Report, and its postings balance
  - Online POS transaction latency is measured within the < 2 s budget — §6 metric 6
  - Credit limits are enforced on order commitments and exposure is itemised
  - The AR subledger equals the receivables control account
  - Every posting in the cycle is balanced — §6 metric 1
- **Evidence**: The end-to-end run with document ids, the latency report, the AR reconciliation and the Z-Report.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Phase 4 — Manufacturing

### Stream: `BOM` — Bill of Materials & Routing (§2.4)

#### Task ID: T-4.BOM.01
- **Title**: Multi-level BOM with scrap percentage
- **Description**: Model BOMs (header, items, quantities, scrap %) to arbitrary depth, with the ability to explode a finished item into all its levels.
- **Plan source**: §2.4 Core Components (Multi-level Bill of Materials (BOM) with scrap % and routing), §4 Phase 4 bullet 1, §5 BOM + BOM Item
- **Scope**: BOM structure, scrap and explosion. Routing is T-4.BOM.02; costing is T-4.WO.05.
- **Variables / Config**: `scrap_percent`, `uom_conversion_factor`, `company_id`.
- **Dependencies**: T-1.INV.01, T-0.AUDIT.02
- **Acceptance Criteria**: A three-level BOM explodes to the correct total component quantities including scrap at each level; a circular BOM is rejected; a component cannot be its own ancestor; scrap of zero means no uplift and is distinguishable from "unset"; a BOM in use by a work order cannot be silently changed (a new version is required).
- **Evidence**: A three-level explosion with per-level scrap, plus a circular-BOM rejection.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: TODO

#### Task ID: T-4.BOM.02
- **Title**: Routing — operations and their sequence
- **Description**: Attach operations in sequence to a BOM, each with its work center, setup and run time, and the components consumed at that operation.
- **Plan source**: §2.4 Core Components (… and routing), §4 Phase 4 bullet 1, §5 Operation
- **Scope**: Routing definition and sequencing. Capacity and rates are T-4.WC.01.
- **Variables / Config**: `operation_sequence`, `uom_conversion_factor`.
- **Dependencies**: T-4.BOM.01
- **Acceptance Criteria**: Operations carry a unique sequence within the BOM; times are expressed per unit and per batch consistently and the basis is labelled; a component can be assigned to a specific operation or to the BOM generally, and both are reported; a routing cannot have a gap or duplicate sequence.
- **Evidence**: A four-operation routing with one operation-specific component, validated and displayed in order.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `WC` — Work Centers & Capacity (§2.4)

#### Task ID: T-4.WC.01
- **Title**: Work centers — capacity, hourly rates, downtime
- **Description**: Model work centers with their available capacity per period, hourly cost rate and downtime allowance.
- **Plan source**: §2.4 Core Components (Work Centers & Operations (capacity, hourly rates, downtime)), §4 Phase 4 bullet 2
- **Scope**: Work center master data. Basic capacity planning is T-4.WC.02; advanced planning is Phase 6.
- **Variables / Config**: `work_center_capacity`, `work_center_hourly_rate`, `downtime_percent`.
- **Dependencies**: T-4.BOM.02
- **Acceptance Criteria**: Capacity is expressed per period and the period is stated; downtime reduces the effective capacity used downstream; hourly rate changes are dated so historical costing is not restated; a capacity or rate of zero is rejected or explicitly flagged as invalid.
- **Evidence**: Two work centers with different capacities/rates and a dated rate change that does not alter a past job's cost.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-4.WC.02
- **Title**: Basic capacity planning
- **Description**: Load planned and open work orders onto work centers over a horizon and show overload/underload per period.
- **Plan source**: §2.4 Key Capabilities (Capacity planning and costing), §4 Phase 4 bullet 2
- **Scope**: Basic load calculation and view over the horizon. Constraint-based levelling and finite scheduling are Phase 6 (T-6.ADV.01).
- **Variables / Config**: `mrp_horizon_days`, `work_center_capacity`, `downtime_percent`.
- **Dependencies**: T-4.WC.01
- **Acceptance Criteria**: Load equals the sum of operation times of the work orders assigned to each work center, adjusted for downtime; overload is visible per period against stated capacity; an unassigned operation is reported rather than ignored; the calculation is reproducible from the work orders.
- **Evidence**: A load chart for a horizon with one overloaded and one underloaded period, reconciled by hand to the source work orders.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `WO` — Shop Floor Control (§2.4)

#### Task ID: T-4.WO.01
- **Title**: Work order creation from BOM and demand
- **Description**: Create work orders for a finished item and quantity, snapshotting the BOM/routing version used and expanding component requirements.
- **Plan source**: §2.4 Core Components (Shop Floor Control (Work Orders …)), §4 Phase 4 bullet 3
- **Scope**: Work order creation and requirement expansion (manual or MRP-suggested). MRP suggestion generation is T-4.MRP.02.
- **Variables / Config**: `scrap_percent`, `uom_conversion_factor`.
- **Dependencies**: T-4.BOM.01, T-4.WC.01
- **Acceptance Criteria**: Requirements equal the BOM explosion for the ordered quantity including scrap at every level; the work order records the exact BOM/routing version so later BOM edits do not change it; a work order without a routing can still be created only if the plan allows (not stated) — otherwise the absence is reported rather than assumed; status transitions are auditable.
- **Evidence**: A work order with its requirement list reconciled to the BOM explosion, and a later BOM edit leaving it unchanged.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-4.WO.02
- **Title**: Job cards, time tracking and production output
- **Description**: Issue job cards per operation, record time booked (setup and run) and produced quantities including rejects, per operator and work center.
- **Plan source**: §2.4 Key Capabilities (Time tracking and production output), §2.4 Core Components (… Job Cards …), §4 Phase 4 bullet 3
- **Scope**: Job card execution and time/output capture. Costing of the captured time is T-4.WO.05.
- **Variables / Config**: `operation_sequence`, `work_center_hourly_rate`.
- **Dependencies**: T-4.WO.01
- **Acceptance Criteria**: Time is booked per operation and cannot exceed a configured limit without acknowledgement; produced and rejected quantities are recorded separately and reconcile to the work order quantity; total booked time equals the sum of job card entries; a job card cannot be edited after the operation is closed except by an audited correction.
- **Evidence**: A work order with two operations and booked time, reconciled to the work order quantity and to the sum of entries.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-4.WO.03
- **Title**: Material issue to work orders
- **Description**: Issue components from stock to a work order, recording the location, quantity, UOM and value, and posting the WIP movement.
- **Plan source**: §2.4 Core Components (… Material Issues …), §4 Phase 4 bullet 3
- **Scope**: Issue and the WIP posting. Substitution/alternate materials are not named in the plan.
- **Variables / Config**: `uom_conversion_factor`, `costing_method` (issued value), `warehouse_hierarchy_levels`.
- **Dependencies**: T-4.WO.01, T-1.INV.05
- **Acceptance Criteria**: Issued quantities are tracked against requirements with the remaining requirement visible; over-issue beyond requirement plus tolerance is refused or explicitly overridden and recorded; the issue creates stock ledger entries and a balanced WIP posting; issued value uses the item's configured costing method.
- **Evidence**: A work order partially issued with remaining requirements shown, plus an over-issue rejection.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-4.WO.04
- **Title**: Finished goods receipt from a work order
- **Description**: Receive the produced item into stock from the work order, consuming the corresponding component quantities and clearing WIP.
- **Plan source**: §2.4 Core Components (… Finished Goods), §4 Phase 4 bullet 3
- **Scope**: Production receipt and its stock effect. Cost calculation is T-4.WO.05.
- **Variables / Config**: `uom_conversion_factor`, `costing_method`, `warehouse_hierarchy_levels`.
- **Dependencies**: T-4.WO.03
- **Acceptance Criteria**: Receiving the full quantity consumes exactly the BOM requirement (within tolerance) and clears the work order's WIP to zero; partial receipts consume proportionally and leave the balance consistent; a receipt that would leave unreconciled WIP is reported rather than silently accepted; every movement appears in the stock ledger.
- **Evidence**: One partial and one final receipt with WIP cleared to zero and the stock ledger shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-4.WO.05
- **Title**: Production costing and its GL posting
- **Description**: Cost a work order from issued material value plus operation time at work center rates, compare to the standard/expected cost, and post the variance.
- **Plan source**: §2.4 Key Capabilities (Capacity planning and costing), §4 Phase 4 exit criteria (correct costing)
- **Scope**: Cost roll-up, variance and posting. No overhead allocation models beyond the named hourly rates (not named in the plan).
- **Variables / Config**: `work_center_hourly_rate`, `costing_method`, `scrap_percent`.
- **Dependencies**: T-4.WO.04, T-4.WC.01, T-1.ACCT.03
- **Acceptance Criteria**: Work order cost equals material issued value plus booked time at the recorded (dated) rates; scrap quantity is attributed to cost, not lost; the variance against expected is computed and posted, and the posting balances; recosting the same work order twice produces no duplicate posting; the finished goods value equals the work order cost.
- **Evidence**: A completed work order costed by hand-checked arithmetic, with the variance posted and the finished goods value agreeing.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: TODO

### Stream: `MRP` — Material Requirements Planning (§2.4)

#### Task ID: T-4.MRP.01
- **Title**: MRP engine — net requirement calculation (sales orders vs stock)
- **Description**: Compute net requirements by exploding demand (sales orders) against on-hand and open supply, applying BOM levels, scrap and lead time, over the configured horizon.
- **Plan source**: §2.4 Core Components (Material Requirements Planning (MRP) engine), §2.4 Key Capabilities (Net requirement calculation (Sales Orders vs Stock)), §4 Phase 4 bullet 4
- **Scope**: The netting calculation and its output. Converting output to suggested orders is T-4.MRP.02; accuracy verification is T-4.MRP.03.
- **Variables / Config**: `mrp_horizon_days`, `mrp_bucket`, `mrp_demand_sources` (Sales Orders vs Stock), `scrap_percent`.
- **Dependencies**: T-3.SALES.04, T-1.INV.04, T-4.BOM.01
- **Acceptance Criteria**: Net requirement equals demand minus available stock and open supply at every BOM level, with scrap applied and lead times respected across buckets; a shortage at a sub-level propagates to the parent requirement; running MRP twice on unchanged data produces identical results (deterministic); the calculation states its inputs (horizon, bucket, demand sources) on every run.
- **Evidence**: A hand-checkable dataset (two levels, one shortage) whose net requirement matches the manual calculation, plus a repeat run producing identical output.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: TODO

#### Task ID: T-4.MRP.02
- **Title**: MRP output to planned orders and purchase requisitions
- **Description**: Turn net requirements into suggested production and purchase actions, and let a planner convert a suggestion into a work order or a purchase requisition.
- **Plan source**: §2.4 Core Components (MRP engine), §2.3 Core Components (Purchase Requisitions), §4 Phase 4 bullet 4
- **Scope**: Suggestion generation and conversion. Automatic release without a planner is not named in the plan and is not built.
- **Variables / Config**: `mrp_horizon_days`, `mrp_bucket`, `approval_levels` (converted requisitions inherit approval).
- **Dependencies**: T-4.MRP.01, T-2.PROC.02
- **Acceptance Criteria**: Every net requirement produces exactly one suggestion with a quantity, date and type (produce or purchase), and the suggested quantity equals the net requirement; converting a suggestion creates the target document and marks the suggestion consumed so it cannot be converted twice; a stale suggestion (inputs changed since the run) is flagged before conversion.
- **Evidence**: One MRP run converted into one work order and one purchase requisition, with a second conversion attempt refused.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-4.MRP.03
- **Title**: MRP net requirement accuracy verification
- **Description**: Verify the MRP engine's net requirements against independently calculated expectations and record the accuracy result that §6 metric 4 and the Phase 4 gate depend on.
- **Plan source**: §6 metric 4 (MRP net requirement accuracy — target set 2026-09-17: 100 % exact), §4 Phase 4 exit criteria
- **Scope**: Verification only — measurement and reporting of differences; corrections are new findings for the owning tasks.
- **Variables / Config**: `mrp_horizon_days`, `mrp_bucket`, `scrap_percent`.
- **Dependencies**: T-4.MRP.01, T-4.MRP.02
- **Acceptance Criteria**: Expected net requirements are calculated independently (by hand or a separate method) for a dataset that exercises multi-level BOMs, scrap, partial stock and open supply; the engine's output must match **exactly — 100 %, any difference is a defect** (target set 2026-09-17); the dataset and the full comparison are recorded.
- **Evidence**: The comparison table (expected vs engine, per item per bucket) showing a 100 % match, and any difference raised as a defect rather than explained away.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-4.X.GATE
- **Title**: Phase 4 exit check
- **Description**: Verify the plan's Phase 4 exit criteria against the running system.
- **Plan source**: §4 Phase 4 Exit Criteria (verbatim): *Ability to produce finished goods from raw materials with correct costing.*
- **Scope**: Verification only.
- **Variables / Config**: `costing_method`, `work_center_hourly_rate`, `scrap_percent`.
- **Dependencies**: T-4.BOM.01, T-4.BOM.02, T-4.WC.01, T-4.WC.02, T-4.WO.01, T-4.WO.02, T-4.WO.03, T-4.WO.04, T-4.WO.05, T-4.MRP.01, T-4.MRP.02, T-4.MRP.03
- **Acceptance Criteria**:
  - A work order is created from a multi-level BOM, materials are issued, time is booked, finished goods are received, and WIP clears to zero
  - The produced item's cost equals material value plus booked operation time at dated rates, and agrees with the finished goods stock value
  - Scrap is attributed to cost rather than lost
  - MRP net requirements match independently computed expectations **exactly** on a multi-level dataset (100 %, any difference a defect) — §6 metric 4
  - A basic capacity view shows overload/underload per period from real work orders
  - Every posting in the cycle is balanced — §6 metric 1
- **Evidence**: The end-to-end production run with document ids, the cost reconciliation, the MRP accuracy comparison and the capacity view.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Phase 5 — HR & Payroll

### Stream: `EMP` — Employee Master & Org (§2.6)

#### Task ID: T-5.EMP.01
- **Title**: Employee master — contracts and assigned assets
- **Description**: Build the employee role on the shared Party master with employment details, contract terms and history, and the assets assigned to the employee.
- **Plan source**: §2.6 Core Components (Employee master (contracts, assets, org chart)), §4 Phase 5 bullet 1, §5 Employee
- **Scope**: Employee identity, contracts and assigned assets. Org structure is T-5.EMP.02; lifecycle movements are T-5.EMP.03.
- **Variables / Config**: `company_id`, `payroll_cycle` (employment terms feed it), `rbac_roles` (restricted personal data).
- **Dependencies**: T-0.PARTY.01, T-0.AUDIT.02, T-0.SEC.01
- **Acceptance Criteria**: An employee is created from a party with the employee role and can also hold other roles without duplication; contract history preserves previous terms (a new contract supersedes rather than overwrites); assigned assets are trackable to a return date; salary and personal fields are restricted by field-level permissions and never returned to an unauthorised role.
- **Evidence**: An employee with two contract revisions and one assigned asset, plus a permission check showing salary withheld from an unauthorised role.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.EMP.02
- **Title**: Org chart (hierarchical reporting lines)
- **Description**: Model the reporting hierarchy over employees and expose it as an org chart with manager relationships and cost centre/department grouping.
- **Plan source**: §2.6 Core Components (… org chart), §1 principle 4 (hierarchical master data), §4 Phase 5 bullet 1
- **Scope**: Reporting hierarchy and its view. Approval routing may consume it where configured, but leave approval is T-5.LEAVE.03.
- **Variables / Config**: `company_id`; departments/cost centres (not named in the plan beyond the hierarchy).
- **Dependencies**: T-5.EMP.01
- **Acceptance Criteria**: Reporting lines form a tree with one root per company and no cycles; a manager cannot report to their own descendant; an employee's reporting line is dated so historical structure is retrievable; the chart renders the hierarchy including employees with no manager.
- **Evidence**: A multi-level hierarchy rendered as a chart, plus a rejected circular reporting line.
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: TODO

#### Task ID: T-5.EMP.03
- **Title**: Employee lifecycle movements
- **Description**: Handle joining, transfers (department/location/reporting line), promotion/salary revision and exit, with effective dates and the downstream effects on payroll and attendance eligibility.
- **Plan source**: §2.6 Key Capabilities (End-to-end employee lifecycle), §2.6 Core Components (Employee master (contracts …))
- **Scope**: Lifecycle movements and their effective dating. Payroll itself is T-5.PAY.02.
- **Variables / Config**: `payroll_cutoff_day`, `company_id`.
- **Dependencies**: T-5.EMP.01
- **Acceptance Criteria**: Each movement has an effective date and does not rewrite history; a mid-period movement affects payroll only for the correct portion of the period; an exited employee stops accruing leave and cannot be included in a new payroll run; movements are auditable with actor and reason.
- **Evidence**: One mid-month transfer and one exit, with the payroll-period effect and the leave accrual stopped.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `ATT` — Attendance & Shifts (§2.6)

#### Task ID: T-5.ATT.01
- **Title**: Shift management
- **Description**: Define shifts (start, end, breaks, grace) and roster employees onto them, supporting repeats and overrides.
- **Plan source**: §2.6 Core Components (Attendance & Shift Management (biometric integration, overtime, late tracking)), §4 Phase 5 bullet 2
- **Scope**: Shift definitions and rosters. Device capture is T-5.ATT.02; biometric device integration is Phase 6.
- **Variables / Config**: `shift_definitions`, `roster_cycle`, `late_grace_minutes`, `holiday_calendar`.
- **Dependencies**: T-5.EMP.02
- **Acceptance Criteria**: A roster resolves to exactly one shift per employee per day (or none); overlapping shifts are rejected or flagged; a roster override records who changed it and why; the roster respects the holiday calendar; a shift change does not retroactively alter already-processed attendance without an audited correction.
- **Evidence**: A roster week with an override and a holiday, with the resolved shift per day shown.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.ATT.02
- **Title**: Attendance log capture
- **Description**: Record attendance events (in/out, per day) from manual entry and from an integration point that a device feed can later use, and derive worked hours from them.
- **Plan source**: §2.6 Core Components (Attendance & Shift Management (biometric integration …)), §4 Phase 5 bullet 2, §5 Attendance Log
- **Scope**: Capture and derivation, including the ingestion contract for devices. The device adapter itself is Phase 6 (T-6.OFFLINE.02).
- **Variables / Config**: `attendance_source` (`manual` now; `biometric device` in Phase 6), `shift_definitions`, `late_grace_minutes`.
- **Dependencies**: T-5.ATT.01
- **Acceptance Criteria**: Worked hours derive from the matched in/out pair against the rostered shift, with the derivation shown; a duplicate device event (same employee, timestamp, direction) is not double-counted; an unmatched punch is reported rather than dropped; manual correction requires a reason and is audited.
- **Evidence**: A day with a normal pair, an unmatched punch and a manual correction, showing derived hours and both exception cases.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.ATT.03
- **Title**: Overtime and late tracking rules
- **Description**: Apply configurable rules to classify overtime (by threshold and day type) and lateness (by grace), producing the figures payroll consumes.
- **Plan source**: §2.6 Core Components (… overtime, late tracking), §2.6 Key Capabilities (Automated statutory compliance calculations)
- **Scope**: Classification and the resulting quantities/hours. Their monetary effect is T-5.PAY.02.
- **Variables / Config**: `overtime_rules`, `late_grace_minutes`, `shift_definitions`, `holiday_calendar`.
- **Dependencies**: T-5.ATT.02
- **Acceptance Criteria**: Overtime is separated into the configured bands (e.g. normal-day vs rest-day) with the band shown per occurrence; late minutes respect the grace period and do not count as absence; holiday work is classified according to the calendar; the rules are configurable without code change and rule changes are dated so past periods are not restated.
- **Evidence**: A week containing normal overtime, a rest-day overtime and a late arrival, each classified with its band.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `LEAVE` — Leave Management (§2.6)

#### Task ID: T-5.LEAVE.01
- **Title**: Holiday calendar
- **Description**: Maintain the holiday calendar per country/region and year, including the effect of a holiday on working and payroll days.
- **Plan source**: §2.6 Core Components (Leave Management (multi-type, accrual, holiday calendar)), §3 row "Localization"
- **Scope**: Calendar definition and consumption by attendance/leave/payroll. No shift-swap rules (not named).
- **Variables / Config**: `holiday_calendar` (from localization pack), `company_id`.
- **Dependencies**: T-5.EMP.01
- **Acceptance Criteria**: Holidays are defined per year and region and can be seeded from the localization pack; a day's status (working/holiday) is resolved deterministically for attendance, leave and payroll; changing the calendar for a closed payroll period is prevented or explicitly audited.
- **Evidence**: A calendar seeded from a pack and a payroll period resolving working versus holiday days correctly.
- **Estimated Effort**: S
- **Owner Role**: Domain Analyst (Payroll/Statutory)
- **Status**: TODO

#### Task ID: T-5.LEAVE.02
- **Title**: Leave types and accrual rules
- **Description**: Define the multiple leave types and their accrual rules (frequency, entitlement, carry-forward cap), and maintain each employee's balances.
- **Plan source**: §2.6 Core Components (Leave Management (multi-type, accrual, holiday calendar)), §4 Phase 5 bullet 2
- **Scope**: Types, accrual and balances. Application and approval are T-5.LEAVE.03.
- **Variables / Config**: `leave_types`, `leave_accrual_rule`, `holiday_calendar`, `company_id`.
- **Dependencies**: T-5.EMP.01, T-5.LEAVE.01
- **Acceptance Criteria**: Accrual posts on the configured cadence and a re-run for the same period does not double-accrue; a carry-forward cap is enforced at the year boundary with the expiry recorded; a balance is always derivable from accruals plus approved leave (no hand-edited balances); unpaid leave types are distinguishable from paid ones for payroll.
- **Evidence**: One accrual cycle with a carry-forward capped case, and a balance reconciled to its underlying entries.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.LEAVE.03
- **Title**: Leave application, approval workflow and balances
- **Description**: Let employees apply for leave, route applications through the configured approval chain, and reflect approved leave in balances and attendance.
- **Plan source**: §2.6 Core Components (Leave Management), §3 row "Workflows & Approvals", §4 Phase 5 bullet 2
- **Scope**: Application, approval and the resulting balance/absence effect. Payroll's consumption of it is T-5.PAY.02.
- **Variables / Config**: `leave_types`, `approval_levels`, `approval_thresholds`, `holiday_calendar`.
- **Dependencies**: T-5.LEAVE.02, T-0.WF.01
- **Acceptance Criteria**: An application exceeding the available balance is refused or requires an explicit, audited override; approval deducts the balance exactly once (a duplicate approval is idempotent); a rejected or cancelled application restores nothing incorrectly; approved leave marks the days as leave in attendance; the approval chain follows configuration and the decision history is complete.
- **Evidence**: One approved, one rejected and one over-balance application, with balances and attendance for the approved case.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `PAY` — Payroll (§2.6)

#### Task ID: T-5.PAY.01
- **Title**: Payroll structure — earnings/deductions and statutory rules
- **Description**: Define the payroll component structure (earnings and deductions, taxable flags, order) and load the statutory deduction rules from the active localization pack.
- **Plan source**: §2.6 Core Components (Payroll Engine (… statutory deductions, loans, payslips)), §3 row "Localization"
- **Scope**: Structure and statutory rule configuration. Calculation is T-5.PAY.02; statutory reporting is T-5.PAY.04.
- **Variables / Config**: `payroll_components`, `statutory_pack`, `tax_pack`, `payroll_cycle`, `base_currency`.
- **Dependencies**: T-5.EMP.01, T-0.LOC.01
- **Acceptance Criteria**: Components are configurable per company without code change and carry a defined calculation basis and order; statutory deductions come from the pack and no rate is hard-coded in application code; the structure marks each component's taxable status; a pack update changes future runs only, leaving computed periods untouched.
- **Evidence**: One company configured from a pack, with a pack rule change leaving a previously computed period unchanged.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Payroll/Statutory) + Backend Engineer
- **Status**: TODO

#### Task ID: T-5.PAY.02
- **Title**: Payroll engine with attendance and leave linkage
- **Description**: Compute a payroll run per period from the employee's contract, attendance (worked days, overtime, lateness) and leave (paid/unpaid), producing payable earnings and deductions.
- **Plan source**: §2.6 Core Components (Payroll Engine (attendance linkage, statutory deductions, loans, payslips)), §2.6 Key Capabilities, §4 Phase 5 bullet 3 and exit criteria, §5 Payroll Entry / Payslip
- **Scope**: The engine and its run lifecycle (draft → computed → approved). Loans are T-5.PAY.03; payslip output is T-5.PAY.05; posting is T-5.PAY.07.
- **Variables / Config**: `payroll_cycle`, `payroll_cutoff_day`, `overtime_rules`, `leave_types`, `payroll_components`, `holiday_calendar`.
- **Dependencies**: T-5.PAY.01, T-5.ATT.03, T-5.LEAVE.03
- **Acceptance Criteria**: Worked days, overtime, unpaid leave and lateness from attendance/leave feed the calculation and each input is traceable to its source record; a recomputation of the same period on unchanged inputs produces identical results; a closed/approved run cannot be recomputed without an audited correction; an employee with incomplete attendance is flagged rather than silently paid a default; the run states the period, cutoff and pack version used.
- **Evidence**: A full-period run for several employees, with each figure traced to attendance/leave, plus a flagged incomplete-attendance employee.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Payroll/Statutory)
- **Status**: TODO

#### Task ID: T-5.PAY.03
- **Title**: Loans and advances with installment recovery
- **Description**: Record employee loans/advances and recover installments through payroll according to the configured policy.
- **Plan source**: §2.6 Core Components (Payroll Engine (… loans …)), §2.6 Key Capabilities
- **Scope**: Loan records, schedules and recovery. Interest modelling beyond simple schedules is not named in the plan.
- **Variables / Config**: `loan_installment_policy`, `payroll_cycle`.
- **Dependencies**: T-5.PAY.02
- **Acceptance Criteria**: Recovery follows the schedule and stops at the outstanding balance (never over-recovers); the policy's maximum recovery percentage is respected, deferring the remainder visibly; a final settlement closes the loan with the balance at zero; recovery is idempotent across recomputations.
- **Evidence**: One loan recovered over periods including a capped period and a final settlement.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.PAY.04
- **Title**: Statutory compliance reports
- **Description**: Produce the statutory reports defined by the localization pack from a payroll run, with the figures reconciling to that run.
- **Plan source**: §2.6 Key Capabilities (Automated statutory compliance calculations), §3 row "Localization"
- **Scope**: Report production from the run and the pack's definitions. Electronic filing with a revenue authority is not named in the plan and is not built.
- **Variables / Config**: `statutory_pack`, `payroll_cycle`, `company_id`.
- **Dependencies**: T-5.PAY.02, T-0.LOC.01
- **Acceptance Criteria**: Each statutory report reconciles exactly to the payroll run it derives from; the report states the pack version and period; reports use the pack's definitions and no market-specific logic is embedded in application code; a missing pack for a market is reported rather than producing an empty report.
- **Evidence**: One period's statutory report reconciled to the run, plus a missing-pack case reported.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Payroll/Statutory)
- **Status**: TODO

#### Task ID: T-5.PAY.05
- **Title**: Payslip generation
- **Description**: Generate a per-employee payslip for a run showing earnings, deductions, employer contributions, net pay and outstanding loan balances, with historical reproducibility.
- **Plan source**: §2.6 Key Capabilities (Payslip generation and banking files), §5 Payslip, §4 Phase 5 bullet 3
- **Scope**: Payslip output (view/print/export). Distribution by email is optional via the integration boundary — the plan names generation, not delivery.
- **Variables / Config**: `payroll_cycle`, `payroll_components`, `base_currency`.
- **Dependencies**: T-5.PAY.02
- **Acceptance Criteria**: Payslip totals equal the payroll run's figures for that employee; a payslip regenerated after a later run reproduces the original figures exactly; net pay equals gross minus deductions; access is restricted so an employee sees only their own and payroll roles see permitted scopes.
- **Evidence**: Payslips for a run reconciled to it, plus a regeneration after a later run producing identical figures.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.PAY.06
- **Title**: Banking files for payroll disbursement
- **Description**: Produce the bank payment file for a payroll run in the format defined by the market/bank pack.
- **Plan source**: §2.6 Key Capabilities (Payslip generation and banking files), §4 Phase 5 bullet 3
- **Scope**: File generation and validation. Bank transmission is not named in the plan.
- **Variables / Config**: `bank_file_format`, `payroll_cycle`, `base_currency`, `integration_endpoints` (if the file is delivered electronically).
- **Dependencies**: T-5.PAY.02, T-0.LOC.01
- **Acceptance Criteria**: The file's total equals the run's net pay total exactly; the format matches the pack's specification and validates against it; employees without valid bank details are excluded and listed, never silently omitted; regenerating the file for the same run produces an identical file.
- **Evidence**: A generated file reconciled to net pay, validated against the format, plus the excluded-employee list.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.PAY.07
- **Title**: Payroll to GL posting
- **Description**: Post an approved payroll run to the GL: salary expense, statutory liabilities, loan recovery and net pay payable, through the shared posting interface.
- **Plan source**: §2.6 Key Capabilities, §3 row "Double-Entry Integrity", §6 metric 1
- **Scope**: The posting of a run. Reconciliation is part of this task's evidence.
- **Variables / Config**: account mapping (expense, statutory payable, loan, net pay), `company_id`, `base_currency`.
- **Dependencies**: T-5.PAY.02, T-1.ACCT.03
- **Acceptance Criteria**: The posting balances and its totals equal the run's gross, deductions and net exactly; posting an already-posted run is refused; a corrected run reverses and reposts rather than editing entries; the statutory liability accounts reconcile to the statutory reports for the same period.
- **Evidence**: One run posted with the entry reconciled to the run figures and to the statutory report, plus a duplicate-post refusal.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

#### Task ID: T-5.PAY.08
- **Title**: Payroll calculation accuracy verification (< 0.1 % error)
- **Description**: Verify payroll results against independently computed expectations and measure the error rate against §6's < 0.1 % target, recording the dataset and any defects.
- **Plan source**: §6 metric 5 (Payroll calculation error rate < 0.1 %), §4 Phase 5 exit criteria
- **Scope**: Verification only. Corrections are new findings for the owning tasks.
- **Variables / Config**: `payroll_cycle`, `payroll_components`, `statutory_pack`.
- **Dependencies**: T-5.PAY.02, T-5.PAY.07
- **Acceptance Criteria**: Expectations are computed independently for a dataset covering overtime, unpaid leave, lateness, loans, a mid-period joiner and an exit; every difference is explained or logged as a defect; the measured error rate is reported against the < 0.1 % target with the dataset size stated.
- **Evidence**: The expected-versus-computed comparison per employee, the measured error rate, and any defects raised.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-5.X.GATE
- **Title**: Phase 5 exit check
- **Description**: Verify the plan's Phase 5 exit criteria against the running system.
- **Plan source**: §4 Phase 5 Exit Criteria (verbatim): *Accurate monthly payroll generation linked to attendance.*
- **Scope**: Verification only.
- **Variables / Config**: `payroll_cycle` (monthly), `payroll_cutoff_day`, `overtime_rules`.
- **Dependencies**: T-5.EMP.01, T-5.EMP.02, T-5.EMP.03, T-5.ATT.01, T-5.ATT.02, T-5.ATT.03, T-5.LEAVE.01, T-5.LEAVE.02, T-5.LEAVE.03, T-5.PAY.01 … T-5.PAY.08
- **Acceptance Criteria**:
  - A monthly payroll run is produced from real attendance and leave, with every figure traceable to its source record
  - The measured payroll error rate is within the < 0.1 % target — §6 metric 5
  - Statutory deductions follow the localization pack, and the statutory reports reconcile to the run
  - Payslips and banking files reconcile exactly to the run's figures
  - The run's GL posting balances and the statutory liabilities reconcile — §6 metric 1
  - An employee's lifecycle movement (mid-period transfer or exit) is reflected correctly for the period
- **Evidence**: The run with document ids, the accuracy comparison, the statutory reports, payslips, the bank file and the posting reconciliation.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Phase 6 — Advanced Features & Hardening

### Stream: `ADV` — Advanced Planning (§2.4, §4 Phase 6)

#### Task ID: T-6.ADV.01
- **Title**: Advanced MRP and capacity planning
- **Description**: Extend planning beyond a single pass: capacity-constrained planning that reflects work centre limits, multi-period levelling, and rescheduling of released suggestions when demand or supply changes.
- **Plan source**: §4 Phase 6 bullet 1 (Advanced MRP / capacity planning), §2.4 Key Capabilities (Capacity planning and costing)
- **Scope**: The advanced planning layer on top of T-4.MRP.01/T-4.WC.02. No finite-scheduling GUI beyond what the plan names.
- **Variables / Config**: `mrp_horizon_days`, `mrp_bucket`, `work_center_capacity`, `downtime_percent`.
- **Dependencies**: T-4.MRP.02, T-4.WC.02
- **Acceptance Criteria**: A plan that violates work centre capacity in the basic view is rescheduled to respect it, and the constraint that caused each move is reported; changing demand re-plans the affected suggestions only (unaffected ones remain stable); the advanced plan reconciles to the basic net requirements for an unconstrained dataset; the output remains deterministic.
- **Evidence**: A constrained dataset planned twice (basic vs advanced) with the constraint report and the stability check.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: TODO

### Stream: `TRACE` — Traceability (§2.2, §4 Phase 6)

> Batch/lot and serial **tracking** moved to Phase 1 (`T-1.INV.08`, `T-1.INV.09`) on 2026-09-17 — it changes the shape of the stock ledger, so retrofitting it later would mean migrating live tables. This stream now holds the cross-module trace **reporting** only, which needs Phase 4 production to be meaningful.

#### Task ID: T-6.TRACE.03
- **Title**: Full track-and-trace reporting (forward and backward)
- **Description**: From a batch or serial, show the full forward chain (where it went: customers, work orders, transfers) and the backward chain (where it came from: supplier, receipt, production order), including recall-style impact.
- **Plan source**: §2.2 Key Capabilities (Full track-and-trace), §4 Phase 6 bullet 2 (amended 2026-09-17 to "Full track-and-trace reporting")
- **Scope**: The trace queries and their output across all modules. Recall workflow automation is not named in the plan.
- **Variables / Config**: `traceability_mode`, `report_schedule`.
- **Dependencies**: T-1.INV.08, T-1.INV.09
- **Acceptance Criteria**: Backward trace from a sold batch reaches its originating supplier receipt or work order; forward trace from a received batch reaches every customer and work order that consumed it; the chain survives intermediate production steps (raw material → finished good); the report is exportable for a given batch/serial and period.
- **Evidence**: One batch traced forward and backward through a production step, exported, with each hop linked to a document.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `PORTAL` — Supplier Portal (§2.3, §4 Phase 6)

#### Task ID: T-6.PORTAL.01
- **Title**: Supplier portal
- **Description**: Give suppliers scoped self-service access: view and respond to RFQs, acknowledge purchase orders, and submit invoices against them.
- **Plan source**: §2.3 Core Components (Request for Quotation (RFQ) & Supplier Portal), §4 Phase 6 bullet 3
- **Scope**: Supplier-facing access to the existing RFQ, PO and AP documents, with supplier-scoped permissions. No marketplace/discovery features (not named).
- **Variables / Config**: `rbac_roles` (supplier scope), `integration_endpoints` (notification), `tax_pack` (submitted invoices).
- **Dependencies**: T-2.PROC.03, T-2.PROC.06, T-2.AP.01
- **Acceptance Criteria**: A supplier sees only its own RFQs, POs and invoices, and no other supplier's data (isolation checked); an RFQ response submitted in the portal lands in the same comparison matrix as an internally captured one (one record, not two); a PO acknowledgement updates the PO's status; a submitted invoice enters the normal match/review path and is not auto-approved.
- **Evidence**: Two suppliers each seeing only their own documents, plus a portal-submitted RFQ response appearing in the existing comparison matrix.
- **Estimated Effort**: L
- **Owner Role**: Full-stack Engineer
- **Status**: TODO

### Stream: `OFFLINE` — Offline POS & Biometric Integration (§2.5, §2.6, §4 Phase 6)

#### Task ID: T-6.OFFLINE.01
- **Title**: Offline-capable POS with synchronisation
- **Description**: Let the POS continue selling without connectivity and synchronise sales, payments and stock movements when it returns, detecting and reporting conflicts.
- **Plan source**: §4 Phase 6 bullet 4 (Offline POS), §1 principle 7 (Offline-capable POS …), §2.5 Core Components (POS (offline-capable …))
- **Scope**: Offline operation and sync for POS, implemented in the POS client with a local queue synchronised to the API (the backend stays the system of record). No offline operation for non-POS modules (not named).
- **Variables / Config**: `pos_mode` (`offline-capable`), `barcode_symbology`, `payment_tender_types`, `pos_latency_budget`.
- **Dependencies**: T-3.POS.05
- **Acceptance Criteria**: A sale completes with no connectivity and is not lost across a restart of the terminal; each queued sale synchronises exactly once (idempotent, no duplicates on retry); stock availability offline does not oversell unless explicitly configured, and any oversell on sync is reported; a long offline period produces a single reconciliation difference report per terminal; the synchronised sale posts the same entries as an online sale.
- **Evidence**: A set of offline sales queued, the terminal restarted, then synchronised — with the duplicate check, the reconciliation difference report and the postings.
- **Estimated Effort**: L
- **Owner Role**: Full-stack Engineer
- **Status**: TODO

#### Task ID: T-6.OFFLINE.02
- **Title**: Biometric attendance device integration
- **Description**: Implement the device adapter that pulls attendance events from biometric devices into the attendance log through the integration boundary, including employee mapping and re-pull safety.
- **Plan source**: §4 Phase 6 bullet 4 (biometric integration), §1 principle 7 (biometric attendance integration points), §2.6 Core Components (biometric integration)
- **Scope**: Device ingestion and mapping. Device administration (firmware, enrolment UI) is out of scope — it is the vendor's tool.
- **Variables / Config**: `integration_endpoints` (device), `attendance_source` (`biometric device`), `shift_definitions`.
- **Dependencies**: T-5.ATT.02, T-0.INT.01
- **Acceptance Criteria**: Events from a device land in the attendance log against the correct employee; a re-pull of the same window does not duplicate events; an event for an unknown device user is parked and reported rather than dropped or mis-assigned; a device that stops reporting is detected and surfaced; the ingestion is logged in the delivery log.
- **Evidence**: A pull containing a normal event, a duplicate window re-pull and an unknown user, with the resulting attendance log and the parked/reported case.
- **Estimated Effort**: M
- **Owner Role**: Integration Engineer
- **Status**: TODO

### Stream: `ANALYTICS` — Analytics & Dashboards (§3, §4 Phase 6)

#### Task ID: T-6.ANALYTICS.01
- **Title**: Advanced analytics and dashboards
- **Description**: Deliver real-time financial and operational dashboards over the existing ledgers and documents, with drill-down to source records and permission-scoped visibility.
- **Plan source**: §3 row "Reporting" (real-time dashboards), §4 Phase 6 bullet 5 (Advanced analytics & dashboards)
- **Scope**: Dashboard content and drill-down. The framework is T-0.REPORT.01; the basic statements are T-1.ACCT.07.
- **Variables / Config**: `rbac_roles`, `company_id`, `report_schedule`.
- **Dependencies**: T-0.REPORT.01, T-1.ACCT.07
- **Acceptance Criteria**: Each dashboard figure traces to a ledger or document record in one drill-down step; dashboard totals reconcile to the corresponding statement or subledger reconciliation for the same period; a role without permission sees neither the figure nor its aggregate; the dashboard reflects current data (no stale overnight snapshot) or states its refresh time.
- **Evidence**: Dashboards for finance and operations with drill-down, each reconciled to its source, plus a permission-restricted view.
- **Estimated Effort**: L
- **Owner Role**: Frontend Engineer
- **Status**: TODO

#### Task ID: T-6.ANALYTICS.02
- **Title**: Scheduled financial and operational report catalogue
- **Description**: Fill the scheduled report catalogue over the modules (financial statements, aging, supplier scorecards, production output, payroll summaries, Z-Reports) with schedules, recipients and permissions.
- **Plan source**: §3 row "Reporting" (scheduled financial & operational reports), §4 Phase 6 bullet 5
- **Scope**: Catalogue, schedules and delivery. The runner is T-0.REPORT.01.
- **Variables / Config**: `report_schedule`, `rbac_roles`, `company_id`, `dunning_channel` (if delivered by email/SMS).
- **Dependencies**: T-0.REPORT.01, T-1.ACCT.07
- **Acceptance Criteria**: Every report in the catalogue is schedulable with recipients and a stated scope; each produced report is scoped to the recipient's company and permissions; a failed run is visible with its error rather than silently skipped; a re-run for the same period does not duplicate delivery.
- **Evidence**: One scheduled financial and one operational report delivered (or delivery logged), plus a failed-run record and the duplicate-delivery check.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: TODO

### Stream: `HARD` — Performance, Security, Localization Hardening (§4 Phase 6)

#### Task ID: T-6.HARD.01
- **Title**: Performance hardening and load verification
- **Description**: Load-test the ledger-heavy paths (posting, stock ledger reads, statement generation, POS checkout, MRP runs) at realistic volumes, fix what fails, and record the results.
- **Plan source**: §4 Phase 6 bullet 6 (Performance … polish)
- **Scope**: Performance work and its verification on the paths named. No architecture rewrite beyond what the measurements justify.
- **Variables / Config**: `pos_latency_budget`, `mrp_horizon_days`, `costing_method`.
- **Dependencies**: T-6.OFFLINE.01, T-6.ADV.01
- **Acceptance Criteria**: Each named path is measured at a stated dataset size and concurrency with the numbers recorded; POS checkout remains within the < 2 s budget at that load — §6 metric 6; any path that fails its target has a recorded finding with its cause; results are reproducible from a documented command.
- **Evidence**: The load-test report per path (dataset, concurrency, result vs target) and the findings raised.
- **Estimated Effort**: L
- **Owner Role**: DevOps Engineer + QA / Test Engineer
- **Status**: TODO

#### Task ID: T-6.HARD.02
- **Title**: Security hardening review
- **Description**: Audit every module's authorisation and data exposure: RBAC and field-level permissions in force, tenant/company and supplier isolation intact, integration credentials protected, and an access review recorded.
- **Plan source**: §4 Phase 6 bullet 6 (… security … polish), §3 row "Security"
- **Scope**: Review and remediation of the named gaps. No penetration-test certification or external audit (not named).
- **Variables / Config**: `rbac_roles`, `field_level_permissions`, `integration_endpoints`.
- **Dependencies**: T-0.SEC.01, T-6.PORTAL.01
- **Acceptance Criteria**: Every module's endpoints are covered by a permission check (no unguarded route); field-level restrictions are enforced on read and write, not just hidden in the UI; cross-company and cross-supplier isolation is verified by test; credentials are not readable from the application's own interfaces or logs; each finding is recorded with its remediation or an explicit accepted risk.
- **Evidence**: The access-review matrix (module × endpoint × permission), the isolation tests, and the findings list with remediation status.
- **Estimated Effort**: L
- **Owner Role**: Security Engineer
- **Status**: TODO

#### Task ID: T-6.HARD.03
- **Title**: Localization polish
- **Description**: Complete and align the localization packs so every supported market's CoA templates, tax rules, statutory rules and statutory reports are consistent and current.
- **Plan source**: §4 Phase 6 bullet 6 (… localization polish), §3 row "Localization"
- **Scope**: Pack completeness, consistency and currency for the confirmed markets. No new market onboarding beyond the confirmed list.
- **Variables / Config**: `coa_template`, `tax_pack`, `statutory_pack`, `holiday_calendar`, `bank_file_format`.
- **Dependencies**: T-0.LOC.01, T-5.PAY.04
- **Acceptance Criteria**: Each confirmed market's pack covers CoA template, tax rules, statutory deductions, statutory reports, holiday calendar and bank file format with no gaps; packs import cleanly into a fresh company; no application code branches on market; a pack version is recorded and the version in force is visible on the documents and reports it affected.
- **Evidence**: A fresh-company import per market, the pack completeness checklist, and the version shown on one affected document.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Payroll/Statutory) + Domain Analyst (Accounting)
- **Status**: TODO

#### Task ID: T-6.HARD.04
- **Title**: Audit trail coverage verification (every master and transaction)
- **Description**: Verify that every master and every transaction change across all six modules is attributable — actor, timestamp and changed values — and that the trail cannot be altered through the application.
- **Plan source**: §6 metric 7 (Full audit trail for every master and transaction change), §3 row "Audit & Immutability"
- **Scope**: Coverage verification across modules. The framework itself is T-0.AUDIT.02.
- **Variables / Config**: `rbac_roles`, `company_id`.
- **Dependencies**: T-0.AUDIT.02, T-6.HARD.02
- **Acceptance Criteria**: Every master entity and every transaction document type has a trail, verified by enumeration rather than sampling (any gap is named); creates, edits and soft deletes are all captured with before/after values; the trail records are immutable through the application's own interfaces; an attempt to modify a trail record is refused and reported.
- **Evidence**: The coverage matrix (entity → trail present, with an exercised create/edit/delete for each) and the refused-alteration case.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

### Phase Exit Gate

#### Task ID: T-6.X.GATE
- **Title**: Phase 6 exit check
- **Description**: Verify the Phase 6 exit criteria, which were **added to plan §4 on 2026-09-17** (the phase previously stated none).
- **Plan source**: §4 Phase 6 Exit Criteria (added 2026-09-17), §6 metrics 1–5 (re-verified)
- **Scope**: Verification only.
- **Variables / Config**: all metrics: `pos_latency_budget`, `three_way_match_target`, `payroll_error_tolerance`, `costing_method` (re-verification inputs).
- **Dependencies**: T-6.ADV.01, T-6.TRACE.03, T-6.PORTAL.01, T-6.OFFLINE.01, T-6.OFFLINE.02, T-6.ANALYTICS.01, T-6.ANALYTICS.02, T-6.HARD.01, T-6.HARD.02, T-6.HARD.03, T-6.HARD.04
- **Acceptance Criteria**:
  - Advanced MRP/capacity produces a capacity-respecting, deterministic plan on a constrained dataset
  - Forward and backward trace from a batch or serial crosses a production step (batch/lot and serial *tracking* is Phase 1 and is verified by `T-1.X.GATE`)
  - A supplier transacts through the portal with verified isolation from other suppliers
  - Offline POS sells without connectivity, synchronises exactly once after restart, and reports reconciliation differences
  - Biometric events ingest without duplication and report unmapped users
  - Dashboards reconcile to their source with one-step drill-down; the scheduled catalogue delivers financial and operational reports
  - Performance is measured on the named paths, and POS online checkout is within the < 2 s budget at load — §6 metric 6
  - The security review covers every module with no unguarded endpoint and verified isolation; every finding is remediated or an accepted risk is recorded
  - Every confirmed market's localization pack is complete and imports cleanly
  - Audit trail coverage is verified for every master and transaction with no gap — §6 metric 7
  - §6 metrics 1, 2, 3, 4 and 5 (balanced postings, valuation vs GL, 3-way match > 95 %, MRP accuracy, payroll error < 0.1 %) are re-measured and still met after the Phase 6 changes
- **Evidence**: Each check with its output; the load-test report; the access-review matrix; the coverage matrix; the re-measured metrics.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: TODO

---

## Open Questions & Conflicts

### Resolved 2026-09-17 (written into plan §1, §4, §6, §7, §8 and reflected in this ledger)

1. **Phase 6 exit criteria** — added to plan §4; `T-6.X.GATE` restates them as checks.
2. **§2 vs §4 on traceability** — batch/lot and serial *tracking* pulled into Phase 1 (`T-1.INV.08`, `T-1.INV.09`) because it changes the shape of the stock ledger; full trace *reporting* stays in Phase 6 (`T-6.TRACE.03`) because it needs Phase 4 production. The Supplier Portal stays in Phase 6 (`T-6.PORTAL.01`).
3. **Target market** — the Philippines.
4. **Five §2 components with no phase** — confirmed as placed (`T-1.INV.06`, `T-3.SALES.02`, `T-5.EMP.01`, `T-5.ATT.03`, `T-4.WO.02`).
5. **Capacity planning split** — confirmed: basic in Phase 4 (`T-4.WC.02`), advanced in Phase 6 (`T-6.ADV.01`).
6. **Shared Party master** — confirmed in Phase 0 (`T-0.PARTY.01`).
7. **Configuration defaults** — the Phase 1 set is decided and is in the Variables table; the rest are deferred to the phase that needs them.
8. **Inter-company transactions** — out of scope; multi-company means data isolation only (now explicit in plan §8).
9. **§6 metric 4** — target set to 100 % exact match on the verification dataset; any difference is a defect.
10. **§7 as Phase 0 scope** — confirmed, including the `T-0.X.GATE` gate.
11. **Stack** — decided: Python + FastAPI + SQLAlchemy, Next.js, PostgreSQL, Postgres-backed queue (library unpinned), PostgreSQL full-text search.
12. **Repository split** — `paws1234/ERPbackend` and `paws1234/ERPfrontend`, created 2026-09-17 as folders inside `ERPV1` (`ERPV1/ERPbackend`, `ERPV1/ERPfrontend`). `ERPV1` itself is the planning repository `paws1234/ERPPLANNING`: it tracks `.github/` and gitignores both app folders, so `ERPV1/ERPbackend` and `ERPV1/ERPfrontend` are never committed to it.

### Still open

1. **Philippines fiscal year start** — needed by the CoA, period locking, the statements and payroll; it must be confirmed with the pack in `T-0.LOC.01` rather than assumed. *(Affects `T-0.LOC.01`, `T-1.ACCT.01`, `T-1.ACCT.04`, `T-1.ACCT.07`.)*
2. **Job-queue library** — Postgres-backed is decided; `pgqueuer` vs `procrastinate` is not pinned. *(Affects `T-0.REPORT.01`, `T-0.CICD.01`, `T-3.AR.03`, `T-3.AR.04`, `T-6.OFFLINE.01`, `T-6.OFFLINE.02`.)*
3. **Remaining configuration defaults** — deferred by decision 7b to the phase that needs each: `fx_rate_source`, `approval_thresholds`, `three_way_match_tolerance`, aging buckets, `dunning_levels`, `credit_check_mode`, `mrp_horizon_days` / `mrp_bucket`, `payroll_cutoff_day`, `shift_definitions`, `overtime_rules`, `leave_accrual_rule`, supplier scoring weights, `report_schedule`.
4. **Deployment target for the container stack** — the plan fixes one container per service and a single compose file, but not *where* they run (a single host, or a managed database plus app hosts). Compose covers local development and a single-host deploy; anything beyond that needs a decision before `T-0.DEPLOY.03` is used in anger. *(Affects `T-0.DEPLOY.03`, `T-0.CICD.01`, `T-6.HARD.01`.)*
5. **Home of the compose/deployment stack** — narrowed 2026-09-17: only two repositories were created, so the stack file lands in one of them rather than in a third infrastructure repository — most naturally the backend repository, which owns the service definitions. **Confirm which.** *(Affects `T-0.DEPLOY.03`, `T-0.CICD.01`, `T-0.CICD.02`.)*

*(Two items previously listed here are settled as of 2026-09-17: the plan and ledger live in the planning repo `paws1234/ERPPLANNING` (`ERPV1/.github`), and the app repos are `paws1234/ERPbackend` and `paws1234/ERPfrontend` — see the Repositories section.)*

---

*Every task above is `TODO`. Statuses are owned by `/do-task` and must not be changed here.*
