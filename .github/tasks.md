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
| `container_registry` | Where the two service images are published and pulled from | Registry URL + namespace | **`ghcr.io/paws1234`** — public packages (decided 2026-09-18, `T-0.CICD.01`) | `T-0.CICD.01`, `T-0.CICD.02`, `T-0.DEPLOY.03` |
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
| Planning (this one) | `ERPV1/.` | `https://github.com/paws1234/ERPPLANNING.git` | `main` @ `da2c548` — tracks `.github/` (this ledger, the plan, the skills) and the root `.gitignore`; Phase 0 merged except the batch in review |
| Backend | `ERPV1/ERPbackend` | `https://github.com/paws1234/ERPbackend.git` | `main` @ `b36e79a` — the app, its image, the compose stack and its pipeline (`.github/workflows/backend.yml`); 19 tagged commits of Phase 0 work |
| Frontend | `ERPV1/ERPfrontend` | `https://github.com/paws1234/ERPfrontend.git` | `main` @ `da7e483` — the Next.js shell, the typed client and its pipeline (`.github/workflows/frontend.yml`) |

All three remotes are **public**. `ERPbackend/` and `ERPfrontend/` are **gitignored** in the planning repo — `git check-ignore` confirms both — so they are never committed here, not even as gitlinks. The apps' first commits belong to the Phase 0 packaging tasks (`T-0.DEPLOY.01`, `T-0.DEPLOY.02`). Being public, no `.env`, credential or secret may ever be committed to any of the three.

The planning repo tracks `.github/` (this ledger, the plan, the skills) plus the root `.gitignore`. A `/do-task` run starts from the workspace root and lands its diff in the subfolder named in the map below.

The skills carry this map: `/plan` records the repository layout in the plan, `/execute` emits this `## Repositories` section into the ledger, and `/do-task` reads it before writing — so a run routes backend/API work to `ERPbackend` and frontend/UI work to `ERPfrontend` rather than guessing.

| Repo | Tasks |
|---|---|
| **Backend repository** | every task not listed below |
| **Frontend repository** | `T-0.API.02`, `T-0.DEPLOY.02`, `T-0.CICD.02`, `T-2.PROC.04`, `T-5.EMP.02`, `T-6.ANALYTICS.01` |
| **Both** — client in the frontend repo, API and postings in the backend repo | `T-3.SALES.02`, `T-3.POS.01`, `T-3.POS.02`, `T-3.POS.03`, `T-3.POS.04`, `T-6.PORTAL.01`, `T-6.OFFLINE.01`; the verification tasks (`T-0.X.GATE` … `T-6.X.GATE`, `T-6.HARD.01`, `T-6.HARD.02`, `T-6.HARD.04`) span both repos |
| **Deployment stack** — `T-0.DEPLOY.03` | the **backend repository** (`docker-compose.yml` + `docker-compose.dev.yml` landed there, in the repository the user named for it); Open Question 5's *confirm which* is all that is outstanding |

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
- **Evidence**: `ERPbackend/DOMAIN-MODELS.md` (the contract document, in the backend repository — the ledger's `## Repositories` map places this task there). §3–§7 give every Phase-1 entity of §5 its table, columns with types and nullability, cardinality, invariants and an **input/output payload**: Account (§3, self-referencing tree, five classes, unique code per company, class-mixing reparent refused), Journal Entry / GL Transaction (§4), Item · Item Variant · barcode · UOM Conversion (§5), Warehouse / Location (§6), Stock Ledger Entry (§7). §4 states §2.1's five fields exactly as the plan does — **posting date** on the entry, **account, debit, credit and the party link** on the line — and matches the columns already in `app/ledger/posting.py` (`journal_entry`, `journal_line`, `MONEY = numeric(20,6)`, the two balance triggers), naming `account_id`/`party_id` as the foreign keys `T-1.ACCT.01` and `T-0.PARTY.01` will make of today's string columns. §5.4 fixes UOM semantics unambiguously (factor direction 1 `from` = factor × `to`, per item, no row for the base UOM, composition along a chain rather than assumed transitivity, exact round trip at scale 6, refusal of `factor <= 0` / `from = to` / a duplicate direction / a pair with no path to the base). §6 fixes the four named levels and the parent-type rule, and requires a leaf (`bin`) on every movement. §7 fixes the append-only stock ledger entry, signed quantity and value with never-negative sums, a required `(source_type, source_id)` document, and the `batch_id`/`serial_id` columns — their presence in the entry's shape is the reason traceability moved into Phase 1. §2 states the rules every entity obeys (company dimension and RLS, soft-deleted masters vs append-only ledgers, exact decimals as JSON strings, enums refused rather than defaulted). §8 excludes every Phase 2–6 entity, so nothing beyond Phase 1 is defined; §9 lists the values deferred to their owner (`coa_template` accounts to T-0.LOC.01/T-1.ACCT.01, `costing_method` column to T-1.INV.04, `fiscal_year_start` to the pack, how a request names its company to T-0.API.01). No runnable check: the deliverable is a written contract, not logic — the reason recorded for `T-0.STACK.01` applies unchanged. Used unmodified by `T-1.ACCT.*` and `T-1.INV.*`.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Accounting)
- **Status**: DONE

### Stream: `API`

#### Task ID: T-0.API.01
- **Title**: API conventions and contract publication
- **Description**: Fix the conventions every endpoint follows — URL versioning, one error shape, pagination and filtering, auth claims, and idempotency keys on posting endpoints — and publish the generated OpenAPI contract as a **versioned artifact** that the frontend repository consumes without access to this repository's source.
- **Plan source**: §1 principle "API-first", §3 row "Delivery & Packaging", §7.3
- **Scope**: Cross-cutting conventions, contract publication and the version policy. Per-entity contracts are T-0.MODELS.01 and the phases' own tasks.
- **Variables / Config**: `api_base_url`, `cors_allowed_origins`, `company_id` (how it is supplied on a request), `rbac_roles` (auth claims).
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: The contract is generated from the code rather than hand-maintained, is fetchable from a running container, and is published as a versioned artifact the frontend repository can pull without this repository's source; a breaking change requires a new version instead of mutating the published one; every endpoint returns errors in one shape; a posting endpoint honours an idempotency key so a retried request does not post twice; every documented field states its type and whether it is required.
- **Evidence**: `ERPbackend/app/api.py` — the conventions, as code the contract is generated from: versioned URLs under `/api/v1` (`API_VERSION`), one error shape (`{"error": {code, message, details}}`, produced by handlers for the app's own `ApiError`, for request validation, for Starlette's 404/405 and for any unexpected failure, and documented on the app as `ErrorOut` so the contract states the shape the app really returns rather than FastAPI's default 422 schema), pagination (`limit`/`offset` with `1..200`, answering `items/limit/offset/total`), amounts as exact decimal strings in both directions, identity stated per request (`X-Company-Id` → `scope_to_company`, `X-Actor` → `set_actor`, both marked as the place T-0.SEC.01 puts authenticated claims), and `Idempotency-Key` on the posting endpoint (`ApiIdempotencyKey`: unique per company × key, storing the request fingerprint, status and body in the caller's transaction — a retry with the same key and body replays the first answer with `Idempotent-Replay: true` and posts nothing, the same key with a different body is refused as `idempotency_key_reused`). The contract is generated, never hand-maintained: `tools/publish_contract.py` writes `ERPbackend/contract/v1/openapi.json` from `app.openapi()`, and the version comes from the app, so a version's artifact is written only by the code that declares that version — a breaking change is a new version and a new directory, never an edit of the file the frontend pinned. Check: `ERPbackend/tests/check_api_conventions.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with fastapi --with httpx --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_api_conventions.py`, exit 0), green on all five: what a running instance serves at `/api/v1/openapi.json` equals the committed `contract/v1/openapi.json` as JSON (so an endpoint edited without republishing fails); every one of the 34 documented fields states a type, and every field not in `required` is nullable (so "optional" is never implicit); the one error shape for `422 unbalanced_entry`, `422 invalid_request`, `400 invalid_company` and `404 not_found`; a posting endpoint that posted **once** across a retry (one `journal_entry` row, the second answer identical and marked as a replay) and refused a reused key with a different body; and a list page answering `(limit 1, offset 0, total 2, 1 item)` with `offset` advancing. The artifact is fetchable from a running process at `/api/v1/openapi.json`; the container half of that criterion — the same route served by the packaged process — is verified by `T-0.DEPLOY.01`'s start check.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-0.API.02
- **Title**: Frontend application shell and typed API client
- **Description**: Scaffold the Next.js application in the **frontend repository** as a separate deployable: routing and layout shell, session/auth handling against the API, and a typed client generated from the published contract artifact.
- **Plan source**: §1 principles "API-first", "one container per service" and "separate repositories", §7.3, §7.4
- **Scope**: The shell, the session and the client, in the frontend repository. Feature screens belong to their phases.
- **Variables / Config**: `api_base_url`, `cors_allowed_origins`, `rbac_roles` (what the shell reveals).
- **Dependencies**: T-0.API.01, T-0.SEC.01
- **Acceptance Criteria**: The frontend's calls go through the generated client and it holds no database credentials or direct database connection; the repository builds and typechecks from the published contract artifact alone, with no backend checkout present; a 401 leads into a session flow and a 403 does not break the shell; a contract field removed breaks the build rather than failing at runtime; the app runs as its own process, not embedded in the backend.
- **Evidence**: The **frontend repository** (`ERPfrontend`) — the shell as its own deployable: `package.json` (next/react pinned exactly), `tsconfig.json`, `next.config.mjs` (`output: "standalone"`, so the runtime carries the app and not the toolchain), `app/layout.tsx` (the shell frame), `app/page.tsx` (the ledger screen, rendered per request via `force-dynamic`), `lib/api.ts` (the typed client) and `lib/contract.d.ts` generated from `contract/v1/openapi.json` — a copy of the backend's published artifact, byte for byte. Every call goes through `request()` and every shape it speaks is read out of the generated contract (`components["schemas"]["JournalEntryOut"]`, `paths["/api/v1/journal-entries"]["get"]`), so a field the backend stops sending fails the build rather than the customer's screen; identity is stated per request (`X-Company-Id`, `X-Actor`) because T-0.API.01's boundary reads it from headers today, and `describeFailure` keeps the refusals apart — 401 → `Unauthenticated` → "Sign in required", 403 → `NotPermitted` → "Not permitted". `scripts/pull-contract.mjs` pulls the artifact from a pinned URL (`CONTRACT_URL`) and regenerates the types, so this repository needs no backend checkout. Check: `npm run check` (`ERPfrontend/scripts/check.mjs`), run 2026-09-17, **green on all six**: no `DATABASE_URL`/`postgres://`/`psycopg`/`ERPbackend` reference in `app/` or `lib/` and no database driver in `package.json`; the vendored contract identical to `ERPbackend/contract/v1/openapi.json`; the generated types current (regenerated into a temporary file and compared, so a hand-edited `.d.ts` is caught); `npx tsc --noEmit` clean with no backend checkout present; a `memo` field removed from the generated types **failing** the typecheck on the page that reads it (the types are restored in a `finally`); `next build` succeeding; and the client tests (`node --experimental-strip-types --test tests/client.test.ts`) showing 401 → `Unauthenticated`, 403 → `NotPermitted` with its code, an `ApiFailure` for other statuses, and every request carrying the company and actor. Shown live, too: the backend was started as its own process (uvicorn against the scratch PostgreSQL 16) and alice, the accountant, read the real ledger — `HTTP 200`, `total: 1`, line `1000 debit 250.000000` — while carol, holding no role, was refused with `HTTP 403 {"error":{"code":"forbidden","message":"'carol' may not 'journal.read' (no roles)","details":null}}`; the built shell then ran as a separate process against that API twice from the same build, `ACTOR=alice` rendering "Shell Demo Trading" with the ledger row and `ACTOR=carol` rendering "Not permitted", and `/api/v1/openapi.json` was fetchable from the running process (10 082 bytes). One trap recorded: with `output: "standalone"` the runtime command is `node .next/standalone/server.js` — Next 16 warns that `next start` is not for it — which is what T-0.DEPLOY.02's image uses. **Closed 2026-10-07 (`/do-task-one-go` batch, no diff) — already implemented:** re-verified in the tree on `main` @ `93fd218` rather than taken from this record. `app/` and `lib/` carry no `DATABASE_URL`/`postgres://`/`psycopg`/`ERPbackend` reference and `package.json` no database driver; `cmp` shows `ERPfrontend/contract/v1/openapi.json` byte-identical to `ERPbackend/contract/v1/openapi.json`; `lib/api.ts` defines the distinct `Unauthenticated` (401) and `NotPermitted` (403) types; `next.config.mjs` sets `output: "standalone"`; and `npm run check` was green on all six assertions on that tree. No product code was written for this id: the deliverable already stood in the frontend repository, so the only diff this run produced for it is this ledger record.
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: DONE

### Stream: `AUDIT`

#### Task ID: T-0.AUDIT.01
- **Title**: Append-only history and soft-delete convention
- **Description**: Implement the storage convention that ledgers and stock/GL entries are append-only (corrections are new entries, never updates or deletes), while masters use soft deletes only.
- **Plan source**: §1 principle 2, §3 row "Audit & Immutability"
- **Scope**: The convention and its enforcement. No audit UI, no reporting.
- **Variables / Config**: none (not configurable — the principle is unconditional).
- **Dependencies**: T-0.STACK.01, T-0.CORE.03
- **Acceptance Criteria**: An update or delete against a posted ledger/stock entry is rejected at the storage boundary; master records mark deleted state without removing the row; existing reads exclude soft-deleted masters by default.
- **Evidence**: `ERPbackend/app/audit.py` — the convention, registered by the module that owns each table and enforced in the database. Appending: `append_only(table)` installs a `BEFORE UPDATE OR DELETE` trigger calling `refuse_ledger_change()`, registered for `journal_entry` and `journal_line` in `app/ledger/posting.py`; masters: `deny_hard_delete(table)` installs a `BEFORE DELETE` trigger calling `refuse_master_delete()` and `SoftDeleteMixin.deleted_at` carries the mark, applied to `Company` in `app/company.py`, where `soft_delete(session, master)` is the only writer. The read side is part of the convention rather than each query: a `Session.do_orm_execute` hook adds `with_loader_criteria(SoftDeleteMixin, deleted_at IS NULL)` to every ORM entity SELECT, so a retired master is absent from `Session.get` and from lists unless the caller asks with the `include_soft_deleted=True` execution option. Check: `ERPbackend/tests/check_audit_conventions.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_audit_conventions.py`, exit 0), green on all four: raw SQL `UPDATE journal_line SET debit = 1` refused with `journal_line is append-only: UPDATE is refused; post a correcting entry instead`, `DELETE FROM journal_entry` refused the same way — both with the posting unchanged afterwards (still one entry, debit still 100.000000); an ORM `session.delete(company)` refused with `company is a master: DELETE is refused; mark it deleted_at instead`; `soft_delete` leaving the row in the table with `deleted_at = 2026-09-17`, invisible to `session.get`/`select(count)` and returned only by the `include_soft_deleted=True` opt-in. The three existing checks were re-run green on the same database (`check_posting_invariant.py`, `check_ledger_integrity.py`, `check_company_isolation.py`), so the new triggers changed nothing else. T-1.INV.03 registers the same `append_only` for the stock ledger when it lands.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-0.AUDIT.02
- **Title**: Audit trail framework (who / when / what, every master and transaction)
- **Description**: Record actor, timestamp, before/after values and origin for every master change and every transaction change, so metric 7 of §6 holds across all six modules.
- **Plan source**: §3 row "Audit & Immutability", §6 metric 7
- **Scope**: Capture and read-back of the trail. No audit dashboards (Phase 6), no log shipping.
- **Variables / Config**: `rbac_roles` (actor identity), `company_id`.
- **Dependencies**: T-0.AUDIT.01
- **Acceptance Criteria**: Creating, editing and soft-deleting a master produces a trail row with actor, timestamp and changed values; a posted transaction's creation is traceable to its originating document; the trail cannot be edited through the application's own API.
- **Evidence**: `ERPbackend/app/audit.py` — the trail. `AuditLog` carries the company dimension, so `app/db.py`'s row-level-security policy already decides which companies' trail a session may read. Rows are written by `audit_row_change()`, a `SECURITY DEFINER` trigger function installed on every audited table (every table carrying `company_id`, plus the `company` master) by the metadata `after_create` hook — the table records itself, so no service method can forget to (child tables are excluded on purpose: `journal_line` cannot change without its append-only parent, whose row already names the change). A row records `actor` (from `app.actor`; `'unknown'` when nobody stated one, because a recorded gap is honest and a failed write is not), `occurred_at`, `action` — `insert`, `update`, `delete`, and the named `soft_delete`/`restore` when `deleted_at` flips — `entity`, `entity_id`, `before_values`/`after_values` as `jsonb`, and `origin_type`/`origin_id` from `app.origin_type`/`app.origin_id` (`set_actor`, `set_origin`; T-0.SEC.01 attributes refusals with them and T-0.API.01 sets them per request). The trail is itself append-only (`append_only(AuditLog.__table__)`), and `app/audit.py` exposes no writer — only `read_trail` — so no API built on it can rewrite history. Check: `ERPbackend/tests/check_audit_trail.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_audit_trail.py`, exit 0), the whole exercise performed as an **ordinary application role** (`SET ROLE`) bound to one company, green on all four: the master's creation recorded with actor `'unknown'`, a timestamp and its after-values though the CRM/database was told nothing; the edit recorded by `alice.auditor` with `before_values['name']`/`after_values['name']`; the retirement recorded as `soft_delete` with `deleted_at` null-then-set; the posting's row naming `supplier_invoice SI-2026-0001` as its origin; and `UPDATE audit_log`/`DELETE FROM audit_log` refused with `audit_log is append-only: UPDATE is refused; post a correcting entry instead` (the check asserts the trail was non-empty and that any accepted statement matched rows, so a silent no-op cannot pass for a refusal). One hazard found and handled: a role switch is session state on a *pooled* connection, and an uncommitted `RESET ROLE` is rolled back when the connection returns to the pool — the check commits the reset, otherwise the next connection inherits the application role and the trail's RLS hides the rows under test.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `PARTY`

#### Task ID: T-0.PARTY.01
- **Title**: Shared Party master (Customer / Supplier / Employee)
- **Description**: Model one party identity with the role(s) a party holds, so Supplier (Phase 2), Customer (Phase 3) and Employee (Phase 5) share one record rather than three parallel masters.
- **Plan source**: §5 Party (Customer / Supplier / Employee), §2.2 hierarchical master data principle
- **Scope**: The base party and its roles only. Role-specific attributes (credit limits, bank details, contracts) belong to the owning modules.
- **Variables / Config**: `company_id`; tax identifiers (consumed by `tax_pack` in Phase 2).
- **Dependencies**: T-0.STACK.01, T-0.AUDIT.01
- **Acceptance Criteria**: One party can hold several roles without duplication; role-specific data is not stored on the base record; a party with no role is rejected; parties are soft-deletable, never hard-deleted.
- **Evidence**: `ERPbackend/app/party.py` — `Party` carries the shared identity (`code`, `name`, `tax_id`) and nothing else: no credit limit, no bank details, no employment fields, because those belong to the roles' own phases and putting them on the shared row is how one counterparty becomes three half-filled masters again. `PartyRole` holds the roles, with `unique(party_id, role)` (a role is held once) and `CHECK role IN ('customer','supplier','employee')` — the three §5 names, no default; `party_role` had to be declared in `app/db.py`'s `CHILD_TABLES` and the scoping guard refused the schema until it was, which is the T-0.CORE.03 convention working as intended. `create_party(...)` takes the roles and refuses an empty list or an unknown name with `UnknownRoleError`; the same invariant is enforced at the storage boundary by `party_has_a_role()`, a deferred constraint trigger on `party` and `party_role` shaped like the ledger's balance rule, so a writer that bypasses the helper is refused at COMMIT. `deny_hard_delete(Party.__table__)` makes it a T-0.AUDIT.01 master: retired by `soft_delete`, never removed. Check: `ERPbackend/tests/check_party_roles.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_party_roles.py`, exit 0), green on all five: the base table's column set is asserted to be exactly `{id, company_id, code, name, tax_id, deleted_at}`; ACME created as customer + supplier is **one** `party` row with two `party_role` rows; a repeated role is stored once while `[]`, `['vendor']` and `['customer','vendor']` are refused; a partyless role written by raw SQL is refused at COMMIT with `party … holds no role; a party is a customer, a supplier or an employee`; and the retired party is still in the table with `deleted_at` set, invisible to a fresh session, with `DELETE` refused as `is a master`. Side-fix carried by this task and T-0.WF.01: the checks now reset the scratch **schema** (`DROP SCHEMA public CASCADE`) instead of the tables each file imports — the new `party`/workflow foreign keys to `company` had broken the rebuild in `check_ledger_integrity.py`, `check_posting_invariant.py` and `check_company_isolation.py` when they ran against a database another check had left tables in. All seven checks were then run in sequence against one database: green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `WF`

#### Task ID: T-0.WF.01
- **Title**: Configurable multi-level approval engine
- **Description**: Build the workflow engine that any document type can attach to: ordered approval levels, threshold bands that route a document, approve/reject/return transitions, and a full decision history.
- **Plan source**: §3 row "Workflows & Approvals", §1 principle 6
- **Scope**: The engine and its configuration surface, exercised by PR/PO/payment/leave in later phases. No document-specific approval rules here.
- **Variables / Config**: `approval_levels`, `approval_thresholds`, `rbac_roles`.
- **Dependencies**: T-0.STACK.01, T-0.AUDIT.02
- **Acceptance Criteria**: A document type is configured with levels and thresholds without code change; a document follows exactly the configured chain; rejection and return are recorded with actor and reason; approval history is immutable.
- **Evidence**: `ERPbackend/app/workflow.py` — the engine every phase's documents attach to. `ApprovalWorkflow` is one row per company × document type, `ApprovalLevel` its ordered levels, each naming the amount it applies from (`threshold_amount`) and the role that decides it; `configure(...)` is the whole configuration surface, and it refuses a second chain for a document type. `chain_for(...)` routes a document by amount — the levels whose threshold it reaches, in order — so a small document needs fewer approvals than a large one; `start_approval(...)` opens a request, or returns `None` when the amount reaches no level, and refuses a document type with no chain at all (`NoWorkflowConfigured`) rather than waving it through: "nobody configured it" is not "approved". `decide(...)` appends an `ApprovalDecision` (actor, action, reason) and moves the request on — approve advances a level and finishes only after the last one, reject and return end it, and a decision at a level the document has not reached, from a role the level does not name (`WrongApprover`), or on a finished request (`RequestNotPending`) is refused. The stated role arrives from the caller; T-0.SEC.01 supplies it from the authenticated request, and only refusals of *permission* belong there. `approval_decision` is append-only (T-0.AUDIT.01), so the record of who approved what stands; `approval_level` and `approval_decision` are declared in `app/db.py`'s `CHILD_TABLES` (the scoping guard refused the schema until they were). Check: `ERPbackend/tests/check_approval_engine.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_approval_engine.py`, exit 0), green on all six: two chains configured as data — `purchase_order` and `leave_request`, the second a document type no module in the repository mentions, which is the "no code change" claim — plus `WorkflowAlreadyConfigured` for a second `purchase_order` chain; a `payment_batch` with no chain refused with `no approval chain is configured for payment_batch; configure one before submitting a document that needs approval`; a 500 PO needing no approval against a 1000 threshold; a 50 000 PO waiting at level 1, refusing `director` at level 1 with `level 1 of purchase_order is decided by 'manager', not 'director'`, still `pending` at level 2 after the manager's approval, then `approved` with decisions `[(1,'approve','mara'), (2,'approve','dana')]` in chain order; `reject` without a reason refused, with a reason recorded as `mara: 'no budget this quarter'` and the request ended; `return` recorded with its reason; and `UPDATE`/`DELETE` on `approval_decision` refused as append-only (the check asserts the table was non-empty and that anything accepted matched rows). Side-fix shared with T-0.PARTY.01: the checks now reset the scratch schema instead of per-file table lists.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `SEC`

#### Task ID: T-0.SEC.01
- **Title**: Role-based access control with field-level permissions
- **Description**: Implement roles, permissions, and field-level read/write restrictions, applied uniformly across modules.
- **Plan source**: §3 row "Security"
- **Scope**: Authorisation only. No authentication provider choice beyond what T-0.STACK.01 fixed, no SSO features the plan does not name.
- **Variables / Config**: `rbac_roles`, `field_level_permissions`.
- **Dependencies**: T-0.STACK.01, T-0.AUDIT.02
- **Acceptance Criteria**: A role without a permission is refused at the API boundary (not merely hidden in the UI); a field marked restricted is absent from both read payloads and write acceptance; permission changes take effect without a deploy; every refusal is attributable in the audit trail.
- **Evidence**: `ERPbackend/app/security.py` — authorisation as data: `Role`, `Permission` (a role's capabilities, named `<resource>.<action>` — no enumeration in code), `FieldPermission` (a read/write restriction on one field of one entity, absent meaning unrestricted) and `RoleAssignment` (a subject's roles; several allowed). `require(...)` refuses a caller whose roles do not hold the capability, `readable_fields(...)` removes a restricted field from a payload instead of nulling it (so "may not see" cannot be confused with "not filled in"), `reject_restricted_fields(...)` refuses a write that sets one — considering only the fields actually set, so a default the API supplied is not held against the caller — and `record_refusal(...)` appends the attempt to the audit trail (T-0.AUDIT.02) with the actor, what was attempted and the roles held, committing on the spot because a guard runs before the work it guards and a refusal that vanished with the rollback it caused would be no record at all. `role`, `permission` and `field_permission` are declared in `app/db.py`'s `CHILD_TABLES` where they have no company of their own. Wired at the boundary in `app/api.py`: `company.read` on the company endpoint, `journal.post` and a per-line field check on the posting endpoint, `journal.read` on the ledger list, with read filtering applied to every payload — including a replayed idempotent answer, which is stored whole and filtered per request — and `AccessDenied` answered as `403` in the one error shape. Check: `ERPbackend/tests/check_security.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with fastapi --with httpx --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_security.py`, exit 0), green on all five: carol, holding no role, refused `403 forbidden` with `'carol' may not 'journal.post' (no roles)` and refused `journal.read` too; bob the auditor refused with `(holds auditor)`; alice's permitted posting stored and read back **with** the `party` link; the same line read by bob **without** the `party` key at all (while `account`/`debit`/`credit`/`line_no` remain); a write to a field marked unwritable refused as `'alice' may not write journal_line.party` with the ledger still holding exactly one entry; `grant(auditor, "journal.post")` taking effect on the very next request with nothing restarted; and 4 refusals on the trail, every one naming its actor and the roles held (`[]` for carol, `["auditor"]` for bob), with the ledger holding exactly the two authorised postings.
- **Estimated Effort**: M
- **Owner Role**: Security Engineer
- **Status**: DONE

### Stream: `INT`

#### Task ID: T-0.INT.01
- **Title**: Integration boundary: outbound adapters and inbound webhook receiver
- **Description**: Establish the single boundary through which the platform talks to externals named in §3: payment gateways (inbound webhooks), biometric devices (inbound sync), barcode printers, and email/SMS (outbound). Includes retry, idempotency keys, and a delivery log.
- **Plan source**: §3 row "Integrations"
- **Scope**: The boundary, contracts and delivery log. The gateway-specific, device-specific and printer-specific mappings belong to the phases that use them.
- **Variables / Config**: `integration_endpoints`, `dunning_channel` (as an outbound consumer), `barcode_symbology` (printer consumer).
- **Dependencies**: T-0.STACK.01, T-0.SEC.01
- **Acceptance Criteria**: A duplicated inbound webhook (same idempotency key) is processed once; a failing outbound send is retried and visible in the delivery log; credentials are configuration, not code; a consumer cannot bypass the boundary.
- **Evidence**: `ERPbackend/app/integrations.py` — the boundary every external meets. Inbound: `receive_inbound(...)` stores the delivery under the sender's idempotency key in `inbound_event`, whose `UNIQUE (company_id, source, idempotency_key)` is the guard — a gateway retry or a re-synced device returns the existing event with `duplicate=True` and its handler is **not** called again, so nothing is booked twice; the row is committed before the handler runs, so a handler that fails leaves the event visible instead of losing it to the rollback it caused. Outbound: `send_outbound(...)` is the only sender, writes the `outbound_delivery` log row before the transport is called, retries up to `max_attempts`, and returns with status `sent` (error cleared) or `failed` carrying `last_error` and the attempt count — a send that never succeeded is visible rather than silent. Credentials are configuration: `endpoint_for(channel)` reads the `INTEGRATION_ENDPOINTS` JSON from the environment (raising `UnconfiguredChannel` when it is missing or the channel is absent) and the token is never copied into the log. The transports (`_transports`) are private to this module and registered by the phase that owns the channel (`register_transport`), so no consumer can send without a log row. Check: `ERPbackend/tests/check_integrations.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_integrations.py`, exit 0), green on all four: the endpoint and its token resolved from the environment while the token appears nowhere in the module's source and nowhere in the delivery log (the log keeps the destination and payload); the same webhook twice is one `inbound_event` and one handler call while `evt-2` is a new event; a transport that failed twice succeeded on attempt 3 (`sent`, 3 attempts, no error) and a transport that always fails is logged `failed` with 4 attempts and `Boom: mailbox unavailable`, both rows readable from the log; a channel with no transport refused rather than silently dropped; the deliveries on the audit trail (`read_trail(entity="outbound_delivery")`); and no module outside `integrations.py` reaches `_transports` (8 other modules scanned). A hazard found while writing it: the boundary commits, and the company binding is transaction-scoped (app.db) — so the boundary re-states the company after each commit, otherwise a plain application role would silently update zero rows under row-level security.
- **Estimated Effort**: M
- **Owner Role**: Integration Engineer
- **Status**: DONE

### Stream: `REPORT`

#### Task ID: T-0.REPORT.01
- **Title**: Reporting framework (real-time query layer + scheduled runner)
- **Description**: Provide the read layer used by real-time dashboards and the runner that executes scheduled financial and operational reports, honouring `report_schedule` and permissions.
- **Plan source**: §3 row "Reporting"
- **Scope**: Framework, scheduling and delivery only. The report content itself is built in Phase 1 (financial statements) and Phase 6 (catalogue/dashboards).
- **Variables / Config**: `report_schedule`, `rbac_roles`, `company_id`.
- **Dependencies**: T-0.STACK.01, T-0.SEC.01
- **Acceptance Criteria**: A registered report is scheduled, produced on time, and delivered to its recipients; a report run is scoped to the requester's company and permissions; a failed run is visible rather than silent.
- **Evidence**: `ERPbackend/app/reporting.py` — the frame, not the content (the statements are T-1.ACCT.07, the catalogue and dashboards are Phase 6). `ReportDefinition` is one row per company × report code carrying its capability, its `schedule` and its recipients; `ReportRun` is one row per attempt with `requested_by`, `started_at`/`finished_at`, `ok`/`failed`, the produced payload **or** the error, and the recipients it was handed to. `validate_schedule`/`schedule_matches` implement the standard five-field cron reading (including the rule that two restricted day fields match either), so "it ran on time" is a decidable question, and `register(...)` refuses a malformed expression and a definition with no recipients. `due(...)` selects the definitions for a minute, for one company. `run(...)` asks T-0.SEC.01 for the definition's capability *before* building anything — a caller who may not see the report is refused and the refusal is on the trail — then records the outcome, and on success hands each recipient to T-0.INT.01's `send_outbound`, so delivery (and its retry and logging) belongs to the boundary and a `failed` run is a row rather than a silent absence. Check: `ERPbackend/tests/check_reporting.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_reporting.py`, exit 0), green on all four: a four-field expression refused at registration; `30 6 * * *` selecting the report at 06:30 and selecting nothing at 06:31, and never selecting the other company's definition of the same code; the run recorded `ok` with its produced payload and delivering to both recipients, with both sends in T-0.INT.01's delivery log as `sent`; a caller holding no `report.read` refused with the refusal attributable on the trail (`carol`, `attempted: report.read`); and both failures visible as `failed` runs — a report with no registered builder (`no builder is registered for 'garble_report'…`) and a builder that raised (`BuilderFailed: the ledger is empty`) with its requester and finish time recorded.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `LOC`

#### Task ID: T-0.LOC.01
- **Title**: Localization pack framework and the Philippines pack (CoA templates, tax rules, statutory reports)
- **Description**: Define the pack structure for CoA templates, tax rules and statutory reports, then author the pack for the confirmed market — the **Philippines** (decided 2026-09-17; §7.4 previously left the market unnamed, and plan §8 now records it).
- **Plan source**: §3 row "Localization", §2.1 (CoA localization support), §2.3 (tax compliance), §2.6 (statutory compliance), §7.6, §8
- **Scope**: Pack structure plus one complete pack per confirmed market: CoA template, tax rules, statutory deduction rules, statutory report definitions, holiday calendar, fiscal calendar, bank file format. No module logic.
- **Variables / Config**: `coa_template`, `tax_pack`, `statutory_pack`, `holiday_calendar`, `fiscal_year_start`, `bank_file_format`.
- **Dependencies**: T-0.STACK.01, T-0.CORE.03
- **Acceptance Criteria**: Pack structure is documented and versioned; the Philippines pack has a CoA template that imports cleanly, tax rules covering the documents that market needs, statutory deduction rules, its statutory report definitions, a holiday calendar and its bank file format; the fiscal year start is confirmed with the pack — it is still undecided as of 2026-09-17 and must not be assumed; no market beyond the Philippines is invented.
- **Evidence**: `ERPbackend/app/localization/__init__.py` (the framework) + `app/localization/philippines/pack.json` (the data). A pack is **versioned data, not code**: market, version, currency, `as_of`, the `coa_template`, `tax_rules`, `statutory_rules`, `statutory_reports`, `holidays`, `bank_file_format` and `fiscal_year_start`. `load_pack` validates before anything consumes it — a repeated account code, an unknown class, a parent that is not listed above its child, a rate outside 0–100 %, a statutory rule with no basis or no schedule, an undated or untyped holiday, a report with no authority, a bank format with no columns — each refused with the pack's name and the offending entry. The readers are `coa_template`, `accounts_by_class`, `tax_rules(market, document_type)`, `statutory_rules(market, kind)`, `holidays(market, year)`, `bank_file_format` and `company_template`. **Nothing is assumed on the market's behalf**: the pack carries `fiscal_year_start: null` because plan §8 leaves it open, and `fiscal_year_start(market)` refuses until the pack — or the caller, stating the confirmed month — settles it; that is exactly the value `Company` (T-0.CORE.03) demands, so a company in this market cannot be created by guessing. Check: `ERPbackend/tests/check_localization.py`, run 2026-09-17 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… uv run --with 'sqlalchemy>=2.0' --with 'psycopg[binary]' python tests/check_localization.py`, exit 0), green on all five: `packs() == ["philippines"]`, so no market beyond the confirmed one is invented; the pack loads with **58 accounts** across the five classes (15 asset, 15 liability, 4 equity, 7 income, 17 expense); **six broken copies** of it are each refused with their reason (`account code '1000' is missing or repeated`, `account 1000 has an unknown class 'capital'`, `account 1010 names a parent '9999' that is not above it`, `tax rule 'VAT-OUT-12' has a rate outside 0–100: 112`, `statutory rule 'SSS-EE' states no basis`, `holiday {'date': '', …}`); `fiscal_year_start(MARKET)` is refused with `the fiscal year start is not confirmed in the pack (plan §8 leaves it open as of 2026-09-17)`, `confirmed=1`/`7` return the caller's month and `confirmed=13` is refused; the pack carries 7 tax rules, 9 statutory rules (6 contributions, 2 withholdings, 1 loan), 9 statutory report definitions (BIR 2550M/2550Q/1601-C/1601-EQ/1604-C/2316, SSS R-3, PhilHealth RF-1, Pag-IBIG MCRF), 16 dated holidays and the bank file format; a company is created for the market with the pack's currency and the confirmed month; and the check asserts the shipped pack's `fiscal_year_start` is **still null** afterwards — a test run must not answer a question that belongs to the pack's owner, so the confirmed path is proved with a copy. Two deliberate omissions recorded in the pack itself: statutory rates and thresholds are marked "as stated for this pack version — confirm with the pack" rather than cited as law, and the movable observances (Chinese New Year, Eid'l Fitr, Eid'l Adha) are absent because they are proclaimed yearly, not fixed.
- **Estimated Effort**: L
- **Owner Role**: Domain Analyst (Payroll/Statutory) + Domain Analyst (Accounting)
- **Status**: DONE

### Stream: `DEPLOY`

#### Task ID: T-0.DEPLOY.01
- **Title**: Backend container image
- **Description**: Package the FastAPI application as a buildable image: pinned dependencies, non-root runtime user, configuration entirely from environment variables, no secrets baked in, and a start command that needs no shell wrapper (so signals reach the process).
- **Plan source**: §1 principle "one container per service", §3 row "Delivery & Packaging", §7.4
- **Scope**: The backend image in the backend repository. The database and frontend images are separate tasks.
- **Variables / Config**: database connection settings and `company_id` defaults, supplied by the environment; no secret is read from a file inside the image.
- **Dependencies**: T-0.STACK.01
- **Acceptance Criteria**: The image builds from a clean checkout and starts against a database given only environment variables; it runs as a non-root user; no secret, credential or `.env` file is copied into the image; the app process is PID 1 and stops on SIGTERM without being killed after a timeout; the image is reproducible from a pinned dependency lock.
- **Evidence**: `ERPbackend/Dockerfile` + `requirements.lock` + `.dockerignore` — one container, one service. Base `python:3.12-slim`; the IPv4 preference appended to `/etc/gai.conf` (pypi publishes AAAA records and these containers have no IPv6 route, which stalls `pip install`); the pinned lock installed (`uv pip compile requirements.in -o requirements.lock`, 17 exact pins — the image's whole dependency set); the app copied in; an `erp` user created and `USER 10001:10001`; `CMD ["uvicorn", "app.api:app", "--host", "0.0.0.0", "--port", "8000"]` in **exec form**, so the server is PID 1 and no shell sits in front of it. Configuration is entirely the environment's — `DATABASE_URL` is read at runtime — and `.dockerignore` keeps `.env`, `tests` and `tools` out of the build context altogether. Check: `ERPbackend/tests/check_backend_image.py`, run 2026-09-17 against a scratch PostgreSQL 16 (Docker), green on all five: the image built in 10.2s; `Config.User` is `10001:10001` and `Config.Cmd` is exactly `["uvicorn","app.api:app","--host","0.0.0.0","--port","8000"]`; a `.env` holding a sentinel secret **was** in the build context while it built and that value appears nowhere in the image (`grep -r` over `/app` and `/etc`) with no `.env` file inside it (the sentinel file is deleted in the check's `finally`, and the check asserts it is gone); all 17 installed versions equal the lock; the container started with nothing but `-e DATABASE_URL=…`, answered `/api/v1/health` with `200 {"version":"v1"}`, served the published contract (10 082 bytes) and read the database through `X-Company-Id`/`X-Actor` (returned `IMAGE-CHECK`); and `/proc/1/cmdline` is uvicorn while `docker stop -t 10` returned in **0.36 s with exit code 0** — SIGTERM handled, not waited out.
- **Estimated Effort**: S
- **Owner Role**: DevOps Engineer
- **Status**: DONE

#### Task ID: T-0.DEPLOY.02
- **Title**: Frontend container image
- **Description**: Package the Next.js application as its own image: production build, API base URL injected from the environment at start, and a runtime that does not carry the build toolchain.
- **Plan source**: §1 principle "one container per service", §3 row "Delivery & Packaging", §7.4
- **Scope**: The frontend image in the frontend repository. The backend image is T-0.DEPLOY.01; the database runs the stock PostgreSQL image.
- **Variables / Config**: `api_base_url`, `cors_allowed_origins`.
- **Dependencies**: T-0.DEPLOY.01, T-0.API.02
- **Acceptance Criteria**: The image builds from a clean checkout and serves the app; the API base URL is read from the environment at start, so one image works against a different backend without a rebuild; source-only toolchain and development dependencies are absent from the runtime image; it runs as a non-root user and stops on SIGTERM.
- **Evidence**: The **frontend repository** (`ERPfrontend/Dockerfile`, `.dockerignore`, `scripts/check-image.mjs`) — three stages, so the runtime carries the application and not the toolchain: `deps` (`npm ci` from the committed lock), `build` (`npm run build`, which needs no backend checkout because `lib/contract.d.ts` is committed), and `runtime`, which copies only `.next/standalone` and `.next/static`, sets `NODE_ENV`/`PORT`/`HOSTNAME`, runs as `USER node` and starts with `CMD ["node","server.js"]` in exec form. `.dockerignore` keeps `node_modules`, `.next`, `tests` and `.env` out of the context. Check: `node scripts/check-image.mjs`, run 2026-09-17, green on all six: both images built from their own repositories; `User=node` and `Cmd=["node","server.js"]`; the runtime image carries no `typescript`, `openapi-typescript`, `@types` or `next`/`tsc`/`npm` CLI; a one-off seeding container wrote a company, a role and a posting into the database (`PYTHONPATH=/app`, so the script could import `app.*`); two backends came up on ports 8000 and 8001; the frontend rendered **"Deploy Check Trading"** and `987.00` from the **containerised** backend; the *same image* started again with `API_BASE_URL=http://127.0.0.1:8001` served the second backend with no rebuild — the base URL is read at start; and `docker stop -t 10` returned in **0.08 s** with exit code 143 (ended by SIGTERM), not the kill timeout's 137. Two traps recorded: `docker build -f Dockerfile` resolves the Dockerfile against the **caller's cwd, not the build context**, so building the backend image from this check's directory silently looked for the frontend's Dockerfile until it was passed absolutely; and the check runs every container with `--network host`, so the database, both backends and both frontends are reachable at `127.0.0.1` without depending on port forwarding through the bridge.
- **Estimated Effort**: S
- **Owner Role**: DevOps Engineer
- **Status**: DONE

#### Task ID: T-0.DEPLOY.03
- **Title**: Compose stack — database, backend and frontend as separate services
- **Description**: One compose file bringing up PostgreSQL, the backend and the frontend as separate containers, consuming each application's **published image** (with a development override for local builds), so neither application repository needs its sibling checked out: shared network, health checks, start ordering, persistent database storage, and configuration through environment variables.
- **Plan source**: §1 principle "one container per service", §3 row "Delivery & Packaging", §7.4
- **Scope**: The three named services, plus the worker service added by T-0.REPORT.01 when the first scheduled job exists. Not production orchestration — see Open Questions.
- **Variables / Config**: database credentials and volume, `api_base_url`, `cors_allowed_origins`, service ports.
- **Dependencies**: T-0.DEPLOY.01, T-0.DEPLOY.02, T-0.CORE.03
- **Acceptance Criteria**: One command starts the database, backend and frontend as three separate containers, with the database healthy before the backend starts; the images come from the registry, so the stack comes up with **neither application repository checked out** (separate repositories); the frontend reaches the backend over the compose network and the backend reaches the database, while the frontend has no route to the database (API-first, verified rather than assumed); database data survives a full stack restart; each service's health is observable.
- **Evidence**: `ERPbackend/docker-compose.yml` + `docker-compose.dev.yml` — the stack, in the backend repository (the repository you named for it). Three services: `db` (`postgres:16`, a named `db-data` volume, a `pg_isready` healthcheck, and `POSTGRES_PASSWORD` with **no default** so a stack must be given its password), `backend` and `frontend` from `${BACKEND_IMAGE}`/`${FRONTEND_IMAGE}` (so the published stack needs no application repository checked out), each with its own healthcheck (python `urllib` for the backend, node `fetch` for the frontend) and `depends_on: condition: service_healthy` — the backend waits for a database that *answers*, the frontend for a healthy backend. Two networks on purpose: `edge` (frontend + backend) and `data` (backend + db, with the frontend absent), so "the frontend has no route to the database" is the network's doing rather than a convention. The frontend's `API_BASE_URL` comes from the environment at start. `docker-compose.dev.yml` is the local-build override and the only place a sibling checkout is used. Check: `ERPbackend/tests/check_compose_stack.py`, run 2026-09-17 with `POSTGRES_PASSWORD=…` (Docker), green on all five: both images built from their own repositories, then the stack was brought up **from a temporary directory containing only `docker-compose.yml`** — `docker compose up -d --wait` in **8.4 s**, neither repository present beside it; all three services `running` and `healthy`; the db's `StartedAt` (15:59:26.445) precedes the backend's (15:59:29.083), so the database was healthy before the backend started; from inside the frontend container `fetch('http://backend:8000/api/v1/health')` answered **200** while `fetch('http://db:5432')` failed with **EAI_AGAIN** — the frontend cannot even resolve the database, attempted rather than assumed; the page fetched through the whole stack rendered "Stack Check Trading" and `1234.00`; and after a full `docker compose down` + `up` the same data still rendered. Ceiling recorded honestly: the ledger's `container_registry` variable is **Not stated** (an Open Question), so the images are named by the local tags this check builds (`erpv1-backend:local`, `erpv1-frontend:local`) instead of a registry reference — the property the criterion exists for, that the stack comes up with **neither application repository checked out**, is what the check proves; pinning the registry belongs to `T-0.CICD.01` once that value is decided. One trap: Compose rejected the frontend's healthcheck until its command was quoted — the ternary's `: ` made YAML read the item as a mapping.
- **Estimated Effort**: M
- **Owner Role**: DevOps Engineer
- **Status**: DONE

### Stream: `CICD`

#### Task ID: T-0.CICD.01
- **Title**: Backend repository pipeline
- **Description**: The backend repository's own pipeline: run the T-0.CORE.02 layers on every change, gate on the ledger-integrity check, and publish the backend image.
- **Plan source**: §7.5 (separate repositories, decided 2026-09-17)
- **Scope**: The backend repository's pipeline only; the frontend repository's pipeline is T-0.CICD.02. No environment-specific infrastructure beyond what the stack and packaging decisions require.
- **Variables / Config**: `container_registry` (where the image is published).
- **Dependencies**: T-0.STACK.01, T-0.CORE.02, T-0.DEPLOY.01
- **Acceptance Criteria**: A change runs unit, integration and ledger-integrity checks; a failing check blocks merge; the backend image publishes to the registry tagged with the commit, with the frontend repository absent; a deploy step exists and is reproducible.
- **Evidence**: `ERPbackend/.github/workflows/backend.yml` (the pipeline) + `ERPbackend/requirements-dev.in` / `requirements-dev.lock` (what the checks need on top of the app's own pinned set: `httpx`, which `fastapi`'s `TestClient` drives and the image does not need) + the **Wiring** section of `ERPbackend/TESTING.md`, which no longer says "not yet in CI". Three jobs. **`checks`** starts a `postgres:16` service and runs **every** `tests/check_*.py` — the loop is deliberate, so a check that lands tomorrow is wired in without editing the workflow — then the ledger-integrity gate as its own final step: `TESTING.md`'s run order (unit → integration → integrity) is the job's step order. Two checks are excluded by name, each with its reason in the file: `check_backend_image.py` builds the image the `publish` job builds for real, and `check_compose_stack.py` needs the frontend repository's image too — this pipeline must never need its sibling. **`publish`** needs the green run and pushes `ghcr.io/paws1234/erpv1-backend:<commit sha>` built from this repository's own `Dockerfile`, authenticating with the run's own `secrets.GITHUB_TOKEN` (`packages: write` on that job only — no new secret is stored anywhere); **`deploy`** (manual dispatch) copies `docker-compose.yml` into a temporary directory holding no source and brings the published image up there, so the deploy is reproducible from the registry alone. **Registry decided 2026-09-18: `ghcr.io/paws1234`, public packages** — the value `T-0.CICD.02` and `T-0.DEPLOY.03` also read, now stated in the Variables table instead of "Not stated". Observed runs, all on this repository's Actions: `35245612760` (**push**, green) and `35245737990` (**workflow_dispatch**, green — `checks` 31 s · `publish` 31 s · `deploy` 23 s). Before either, the twelve DB-backed checks and the gate were run **locally in the workflow's own order**, against a scratch PostgreSQL 16 through a venv holding exactly `requirements-dev.lock` — all green, which is how the job's command list was proved before it was pushed. The deliberately red proof is `35245973296` (`workflow_dispatch` on the scratch branch `ci/prove-failing-check`, which adds one `tests/check_deliberate_failure.py` that asserts False): **`checks` = `failure`**, its log ending `AssertionError: T-0.CICD.01: this check must fail the pipeline`, and `publish` and `deploy` both **`skipped`** — so a brand-new check was picked up with no workflow edit, and a red check stops the image. The deploy log shows what was consumed: `erpv1-backend-1  ghcr.io/paws1234/erpv1-backend:d3aa269ce8adea40ef628cef600d2c1f0d077c99  "uvicorn app.api:app…"  Up (healthy)` with `erpv1-db-1` **healthy before the backend started**; the image pulls **anonymously** (`docker manifest inspect` with no credentials succeeds), so the stack can consume it without a registry login. **"A failing check blocks merge" is observed, not inferred**: `main` now requires exactly the `checks (unit · integration · ledger integrity)` context (`strict: false`, `enforce_admins: false`, no review requirement and no push restriction — direct pushes to `main` behave as before), and the throwaway PR opened from the red branch read `MERGEABLE BLOCKED` with `checks = FAILURE`, then was closed. The scratch branch `ci/prove-failing-check` and closed proof PR #6 remain on the remote as the record.
- **Estimated Effort**: M
- **Owner Role**: DevOps Engineer
- **Status**: DONE

#### Task ID: T-0.CICD.02
- **Title**: Frontend repository pipeline
- **Description**: The frontend repository's own pipeline: typecheck and build against the published contract artifact, then publish the frontend image.
- **Plan source**: §7.5 (separate repositories, decided 2026-09-17)
- **Scope**: The frontend repository's pipeline only. It must not require the backend repository to be checked out or built.
- **Variables / Config**: `container_registry`; `api_base_url` must not be baked in at build time (see T-0.DEPLOY.02).
- **Dependencies**: T-0.CICD.01, T-0.API.02, T-0.DEPLOY.02
- **Acceptance Criteria**: A change in the frontend repository typechecks against the published contract and publishes its image with the backend repository absent; a contract change that breaks the client fails this pipeline instead of failing at runtime; the image is tagged with the frontend commit.
- **Evidence**: `ERPfrontend/.github/workflows/frontend.yml` — the pipeline, and this repository's entire diff, because the repository's correctness is already one command. **`checks`** runs on every push and pull request: `actions/setup-node@v7` on Node 22 (the major the image is built on), `npm ci` from the committed lock, then `npm run check` — the six properties `scripts/check.mjs` asserts (no database driver and no backend source; the generated types current for the vendored contract; the typecheck; a field missing from the contract breaking the build; the production build; the client's 401/403 handling). Nothing needs the backend repository: the contract arrives as a **pinned artifact** (`contract/v1/openapi.json`, refreshed by `scripts/pull-contract.mjs`), so CI is where "the backend checkout is absent" is *lived* rather than asserted — the run's own log says `backend checkout not present — the contract copy stands on its own`, `the repository typechecks with no backend checkout present`, and finally `ok — the shell is built from the published contract alone`. **`publish`** needs the green run and pushes `ghcr.io/paws1234/erpv1-frontend:<commit sha>` built from this repository's own `Dockerfile`, authenticating with the run's own `secrets.GITHUB_TOKEN` (`packages: write` on that job only); `api_base_url` is deliberately not baked in — T-0.DEPLOY.02's image reads it at start. There is no `deploy` job, on purpose: the stack file lives in the backend repository (Open Question 5's narrowing), so this pipeline ends at its published image. Observed runs: `35247288089` (**push**, green — `checks` 25 s), `35247366799` (**workflow_dispatch**, green — `checks` 33 s · `publish` 45 s, tag `e0dda9e5148380f2462e6b77db3f9f447fbbfe09`, which pulls **anonymously**), and the deliberately breaking one `35247590159` (**push of the scratch branch `ci/prove-stale-contract`**, where the contract's `JournalEntryOut` loses the `memo` field and `lib/contract.d.ts` is regenerated to match it, exactly as `npm run contract:pull` would leave them): **`checks` = `failure`** — `2 problem(s)`, the log ending `app/page.tsx(89,30): error TS2339: Property 'memo' does not exist on type '{ company_id: string; … }'` — and `publish` **`skipped`**; the typecheck itself caught it, not a runtime screen. `npm run check` was also run locally, green, before the first push. One trap worth keeping: `gh workflow run` answered `HTTP 404: workflow frontend.yml not found on the default branch` until the first push had registered the workflow, so a dispatch right after the first push can be a moment too early. The scratch branch `ci/prove-stale-contract` remains on the remote as the record.
- **Estimated Effort**: M
- **Owner Role**: DevOps Engineer
- **Status**: DONE

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
- **Evidence**: Every criterion below was checked **in the tree on 2026-09-18**, on the batch's committed trees (backend `d3aa269`, frontend `e0dda9e`) with a venv holding exactly `requirements-dev.lock` and a scratch PostgreSQL 16, plus the two pipelines' observed runs. **Nothing failed, so nothing was fixed — this task is verification only.**
  - *A written stack decision exists* — `ERPbackend/TECH-STACK.md` §Decisions names all five: Python + FastAPI + SQLAlchemy, Next.js, PostgreSQL, a Postgres-backed queue with no Redis (the one library sub-choice deliberately still open), PostgreSQL full-text search, each with its §3 justification.
  - *CI runs unit, integration and ledger-integrity checks and blocks on failure* — `T-0.CICD.01` and `T-0.CICD.02`, observed: green runs `35245612760`, `35245737990`, `35247288089`, `35247366799`; deliberately red runs `35245973296` (`checks = failure`, `publish`/`deploy` `skipped`) and `35247590159` (`checks = failure`, `publish` `skipped`); and the merge itself blocked — throwaway PR #6 read `MERGEABLE BLOCKED` with the required `checks` context failing. The unit layer has no members yet (Phase 1's pure functions will add the first), and the loop runs whatever `tests/check_*.py` exists, so it is the *pipeline* that is complete, not the layer's contents.
  - *The posting primitive rejects unbalanced entries and is the only entry point* — `tests/check_posting_invariant.py`: `ok — the primitive refused the unbalanced set: entry does not balance: debit 100.00 != credit 90.00` (and the single-line and raw-SQL cases). `JournalEntry(` / `JournalLine(` are constructed only inside `app/ledger/posting.py` (lines 235/241), and `post_journal_entry` is called once in the tree, from `app/api.py`.
  - *Company scoping is enforced; a second company cannot see the first company's data* — `tests/check_company_isolation.py`: `ok — the company dimension is enforced; one company cannot see another's rows`, run as a non-owner role so row-level security is what refuses.
  - *Posted ledger entries cannot be updated or deleted; masters soft-delete only* — `tests/check_audit_conventions.py`: `ok — ledgers are append-only and masters retire by marking, in the database` (raw `UPDATE`/`DELETE` refused; stock entries join the same convention with `T-1.INV.03`).
  - *Every master and transaction change is attributable* — `tests/check_audit_trail.py`: `ok — every master and transaction change is recorded and the record is immutable`.
  - *One party holds multiple roles without duplication* — `tests/check_party_roles.py`: `ok — one party identity holds every role it has, and no role's data on the base row`.
  - *A configured approval chain runs a document end to end* — `tests/check_approval_engine.py`: `ok — the chain is configuration, a document follows it exactly, and its history stands` (two document types configured as data, one of them mentioned by no module in the repository).
  - *Field-level permission denial is enforced at the API boundary* — `tests/check_security.py`: `ok — permissions are enforced at the boundary, field by field, and refusals are recorded`.
  - *Duplicate inbound delivery processed once; failed outbound sends retried and logged* — `tests/check_integrations.py`: `ok — duplicates are processed once, every send is logged, and credentials stay in the environment`.
  - *A scheduled report runs, is scoped by permission, and records failure* — `tests/check_reporting.py`: `ok — reports are scheduled, scoped and delivered, and a failure is a row`.
  - *At least one complete localization pack imports cleanly* — `tests/check_localization.py`: `ok — the pack structure is validated and the Philippines pack is complete` (58 accounts; six deliberately broken copies each refused).
  - *Phase 1 domain models and API contracts are complete for the seven entities* — `ERPbackend/DOMAIN-MODELS.md` §3 Account, §4 Journal Entry / GL Transaction, §5 Item · Item Variant · UOM Conversion · barcode, §6 Warehouse / Location, §7 Stock Ledger Entry, with §2's rules (company dimension and RLS, soft-deleted masters vs append-only ledgers, exact decimals, enums refused) and §8 excluding every later-phase entity. No runnable check: a contract document is not logic.
  - *The contract is published from the backend and consumed by the frontend through a generated client, holding no direct database access* — `tests/check_api_conventions.py`: `ok — the conventions hold and the published contract is what the app serves`; frontend `npm run check`: `ok — the shell is built from the published contract alone`, whose CI log reads `backend checkout not present — the contract copy stands on its own`.
  - *One command brings up the database, backend and frontend as three containers, database healthy first, data surviving a restart* — `tests/check_compose_stack.py`: `ok — one command brings up three healthy containers, and only the backend sees the database`; and the backend pipeline's own `deploy` job did the same from the published image against a real database (run `35245737990`, db healthy before the backend started).
  - *Both applications build and publish from their own repositories, contract artifact the only coupling* — `tests/check_backend_image.py`: `ok — the image builds, carries no secret, runs non-root and stops on SIGTERM`; frontend `node scripts/check-image.mjs`: `ok — the frontend image builds, carries no toolchain, and takes its API from the environment`; and both pipelines published (`ghcr.io/paws1234/erpv1-backend:d3aa269ce8adea40ef628cef600d2c1f0d077c99`, `ghcr.io/paws1234/erpv1-frontend:e0dda9e5148380f2462e6b77db3f9f447fbbfe09`) from their own repository with no sibling checked out.
  - *The Open Questions* — the stack is recorded (item 11) and the market confirmed (item 3) as of 2026-09-17; the Phase 1 defaults are decided (item 7); **`container_registry` was answered on 2026-09-18** in this run and is now in the Variables table and the Resolved list, together with the three pieces of ledger drift this gate exposed and corrected. Deliberately still open, and not blockers for Phase 0: the **Philippines fiscal year start** (the pack carries `null` and `T-0.LOC.01` refuses to guess — **the first Phase 1 task, `T-1.ACCT.01`, will need it confirmed**), the job-queue **library**, the per-phase deferred defaults, and the deployment **target**.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: DONE

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
- **Evidence**: `ERPbackend/app/ledger/accounts.py` — the `Account` master (DOMAIN-MODELS.md §3 columns, `class` one of the five, unique `(company_id, code)`) with the tree rules enforced **in the database**: a deferred constraint trigger refuses a child whose class differs from its parent's, a parent in another company, a move under its own descendant (the cycle walk) and retiring a parent that still has live children; `deny_hard_delete` makes it a T-0.AUDIT.01 master. `create_account`, `reparent`, `retire_account`, `account_by_code` and `import_coa_template` are the whole surface (the pack's template imports in its own order, parents first, with no fix-up), and `tree` reads the chart back nesting once per company. Check: `ERPbackend/tests/check_chart_of_accounts.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_chart_of_accounts.py`, exit 0), green on all seven: the Philippines pack's **58 accounts** import in one call with their classes and parents linked; the tree reads back nested (58 nodes, every template parent/child pair intact); a class-mixing child refused by the helper (`account '9999' is asset and its parent '4000' is…`) and by raw SQL at COMMIT (`cannot mix classes`); a cycle refused at COMMIT (`account 1010 cannot be moved under …`); a posted account refused a move into another class; a parent with live children refused retirement by the helper (`account 1010 still has live children…`) and by a raw `UPDATE account SET deleted_at` at COMMIT; a leaf retired by marking (row still in the table, absent from a fresh session's reads, `DELETE` refused as `account is a master`). DOMAIN-MODELS.md §4's link conversion landed here too: `journal_line.account_id` → `account.id` and `party_id` → `party.id` in `app/ledger/posting.py`, resolved from the **code** a line states (`app/ledger/accounts.py`, `app/party.py`) — so the six checks that post now seed the accounts they post to through `ERPbackend/tests/seed.py`. Exposed as `POST /api/v1/accounts`, `GET /api/v1/accounts/tree` and `POST /api/v1/accounts/import-coa`; `contract/v1/openapi.json` republished (53 documented fields, all typed). The whole suite (13 checks) is green on the same tree.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: DONE

#### Task ID: T-1.ACCT.02
- **Title**: General Ledger — immutable journal entry storage
- **Description**: Persist journal entries (posting date, account, debit, credit, party links) as an immutable ledger built on the T-0.CORE.01 primitive, with drill-down to the originating document.
- **Plan source**: §2.1 Core Components (GL – immutable transaction ledger (posting date, account, debit, credit, party links)), §4 Phase 1 bullet 1, §5 Journal Entry / GL Transaction
- **Scope**: Storage, ledger read views and source linking. No statements or reports (T-1.ACCT.07), no period locking (T-1.ACCT.04).
- **Variables / Config**: `company_id`, `base_currency`, `transaction_currency` (recorded, conversion is T-1.ACCT.05), party link.
- **Dependencies**: T-0.CORE.01, T-0.AUDIT.01, T-1.ACCT.01
- **Acceptance Criteria**: Every entry stores posting date, account, debit, credit and party links where applicable; posted entries are append-only (no update, no delete); an entry records the document that produced it; the account balance is derivable from the entries alone.
- **Evidence**: `ERPbackend/app/ledger/posting.py` — `journal_entry.source_type`/`source_id` (DOMAIN-MODELS.md §4's pair, indexed), with `(source_type IS NULL) = (source_id IS NULL)` as a table constraint so **half a pair is refused by the storage boundary** too, and `post_journal_entry(..., source_type=, source_id=)` refusing `IncompleteSourceError` at the caller. `ERPbackend/app/ledger/gl.py` — the read side, derived from the entries and never stored: `ledger_rows` (posting date, account, debit, credit, party link, currency, source per line), `account_balance` (debit-positive sum over the ledger, with an optional period) and `entries_for_source` (the §2.1 drill-down: every posting one document produced). The API carries the pair through (`JournalEntryIn`/`JournalEntryOut`) and `GET /api/v1/journal-entries?source_type=&source_id=` is the drill-down; contract republished. Check: `ERPbackend/tests/check_general_ledger.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_general_ledger.py`, exit 0), green on all six: a document's **two** postings found by its id and a manual entry storing no source; half a pair refused by the primitive (`a posting names its source document as a pair:…`) and by the table at COMMIT (`ck_journal_entry_source_pair`); `UPDATE journal_line` and `DELETE FROM journal_entry` refused as `append-only` with the ledger unchanged; the balance read from the entries equals the same sum computed independently (`300.000000`) while an untouched account answers `0`; the period filter narrows it (`2026-09-17` alone → `100.000000`); and the rows carry the party and the source id. The full suite (14 checks) is green on the same tree, `check_ledger_integrity.py` included.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.ACCT.03
- **Title**: Automatic double-entry posting interface for all sub-modules
- **Description**: Publish the one posting interface that inventory, procurement, sales, POS, manufacturing and payroll call, so no module writes journal lines of its own.
- **Plan source**: §2.1 Key Capabilities (Automatic double-entry posting from all sub-modules), §3 row "Double-Entry Integrity", §6 metric 1
- **Scope**: The posting interface and its account-mapping inputs. Not the per-module call sites (each owning phase wires its own).
- **Variables / Config**: `company_id`; account mapping supplied by the caller.
- **Dependencies**: T-1.ACCT.02, T-0.CORE.02
- **Acceptance Criteria**: Every sub-module posts through this interface (verified by the ledger-integrity check finding no other writer); an unbalanced or single-line call is rejected; postings are atomic with the calling transaction (a rolled-back document leaves no journal entry); the interface is documented for the later phases.
- **Evidence**: `ERPbackend/app/ledger/mapping.py` — the account-mapping half of the interface: company × key → account (`AccountMapping`), `set_mapping` (through T-1.ACCT.01, so a key can only point at an account the company has), `mapped_account` refusing `MissingMappingError` for an unmapped key rather than booking to a guessed account, `mappings` for the settings read, and the module docstring carrying the worked example the later phases follow (`post_journal_entry` + `source_type`/`source_id` + mapped keys). Exposed as `PUT /api/v1/account-mappings/{key}` and `GET /api/v1/account-mappings`; contract republished. Check: `ERPbackend/tests/check_posting_interface.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_posting_interface.py`, exit 0), green on all seven: the tree scan finds **`app/ledger/posting.py` as the only writer** of `journal_entry` / `journal_line` (and the scanner is proved able to fail by scanning an injected writer); a single-line and an unbalanced call refused through the interface; a document that failed after posting left **no** entry behind (atomicity); a key resolved to its account and re-pointing kept one row; an unmapped key refused by name (`no account is mapped for 'cogs' in this company; map…`); a mapping to an account the company does not have refused at the mapping; and T-0.CORE.02's `ledger_gate` green over what the interface wrote. The full suite (15 checks) is green on the same tree.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.ACCT.04
- **Title**: Period locking and posting audit trail
- **Description**: Allow periods to be closed so no further postings (or only controlled ones) land in them, with every lock/unlock and every posting recorded in the audit trail.
- **Plan source**: §2.1 Key Capabilities (Period locking and audit trail)
- **Scope**: Locking, unlocking authority and the posting trail. No year-end closing entries (not named in the plan).
- **Variables / Config**: `period_lock_granularity`, `rbac_roles` (who may lock/unlock), `fiscal_year_start`.
- **Dependencies**: T-1.ACCT.02, T-0.AUDIT.02, T-0.SEC.01
- **Acceptance Criteria**: A posting dated inside a locked period is rejected; unlocking requires a permission and is itself audited with actor and reason; locking does not alter existing entries.
- **Evidence**: `ERPbackend/app/ledger/periods.py` — `AccountingPeriod` (one company × year × month, `state` open/closed, `changed_by`/`changed_at`/`reason` describing the last transition), `lock_period`, `unlock_period` — which asks T-0.SEC.01's `require` for the `period.unlock` capability **and** a reason — and `period_is_locked`. The refusal lives in the **posting primitive** (`post_journal_entry` raises `PeriodLockedError` before it writes anything), so every module is covered by construction rather than each caller remembering to check. Locking writes nothing to the ledger, so existing entries are untouched. Granularity is month, the decided value (`period_lock_granularity`); ponytail comment names the upgrade path to a configurable granularity. Check: `ERPbackend/tests/check_period_locking.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_period_locking.py`, exit 0), green on all six: an open month accepted a posting; the closed month refused a back-dated one (`2026-09 is closed for posting; unlock the period bef…`) with **no row left behind**, while a posting dated in the still-open August landed; the entry posted before the lock is still there with its debit and credit unchanged; unlocking without the capability refused (`'alice.locker' may not 'period.unlock' (no roles)…`) and the month stayed closed; the controller reopened it with a reason and postings were accepted again; and the trail carries both transitions with actor and reason (`('alice.locker', 'insert', 'closed', 'September is reported')`, `('bob.controller', 'update', 'open', 'late supplier invoice for September')`). The full suite (16 checks) is green on the same tree.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.ACCT.05
- **Title**: Multi-currency engine and central FX rate service
- **Description**: Maintain the currency master, hold rates by date, sync daily rates from the configured source, and convert amounts at posting time using the central service.
- **Plan source**: §2.1 Core Components (Multi-Currency Engine – daily FX rate sync), §3 row "Multi-Currency", §4 Phase 1 bullet 2
- **Scope**: Currency master, rate storage, sync job, conversion API. Gain/loss calculation is T-1.ACCT.06.
- **Variables / Config**: `base_currency`, `transaction_currency`, `fx_rate_source`, `fx_sync_schedule`.
- **Dependencies**: T-1.ACCT.02, T-0.INT.01
- **Acceptance Criteria**: Rates are stored per currency pair per date and are never overwritten silently for a past date; the daily sync populates the current day and reports failure loudly; a document in a foreign currency stores its rate, its foreign amount and its base amount; conversion uses the rate for the posting date, not today's.
- **Evidence**: `ERPbackend/app/ledger/currency.py` — the `Currency` master (company-scoped, so a change to it is attributable on the trail rather than a global row nobody owns; soft-deleted like every master), `FxRate` (company × base × currency × date, `rate > 0`), `store_rate` (a past date's rate is refused a rewrite — `HistoricalRateError` — while a re-sync may correct today's, which the trail records), `rate_for` (1 for the base currency; a missing date is refused, never approximated), `convert` (cross rates through the base, quantized to money scale) and the sync: `register_rate_source` + `FX_RATE_SOURCE` configuration + `sync_daily_rates`, which raises `FxSyncFailed` with nothing stored when no source is configured, when the named one is not registered, or when the provider itself fails. The rate reaches the ledger through the **posting primitive**: `journal_entry.exchange_rate` is resolved from the posting date (`post_journal_entry`), so the caller states the currency and never the rate, and the base amount is `amount × exchange_rate`. The API exposes the rate and the derived base amounts on every line (`exchange_rate`, `base_debit`, `base_credit`) plus `POST/GET /api/v1/currencies` and `PUT/GET /api/v1/fx-rates`; contract republished. Check: `ERPbackend/tests/check_multi_currency.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_multi_currency.py`, exit 0), green on all eight: an unregistered currency refused (`'GBP' is not a registered currency for this comp…`); a dated rate stored once (re-storing the same figure changed nothing); a past date's rate refused a rewrite while today's correction landed **on the trail** (`alice.treasury`); a USD posting stored rate `58.5000000000` with base `5850.000000` = `100.00 × 58.5`; the same posting on a day with no stored rate refused (`UnknownRateError`) — proving it is the posting date's rate, not today's; a cross conversion `USD 100.00 → EUR 92.125984` through the base; the sync storing 2 rates from `test-feed`; and a failing sync and an unconfigured sync each raising loudly with **no rows stored**. The full suite (17 checks) is green on the same tree. **Corrected 2026-09-29**: this check pinned `TODAY = date(2026, 9, 19)` and so stopped being true the next morning — `sync_daily_rates(..., on=TODAY)` re-stores a date that `store_rate` now correctly treats as history, and the check failed against code that was right (it was green on the day it was written and red on `main` from 2026-09-20). `TODAY` is now derived from `datetime.now(timezone.utc).date()` and the two relative days follow from it, so the check states what it means on every day it runs. No product code changed: the refusal itself was correct.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.ACCT.06
- **Title**: Realized and unrealized FX gain/loss
- **Description**: Calculate and post unrealized revaluation for open foreign-currency balances at period end, and realized gain/loss on settlement of foreign-currency documents.
- **Plan source**: §2.1 Core Components (unrealized/realized gain/loss), §3 row "Multi-Currency"
- **Scope**: The gain/loss calculations and their postings. No treasury or hedging features (not named).
- **Variables / Config**: `base_currency`, `fx_rate_source`, `period_lock_granularity`.
- **Dependencies**: T-1.ACCT.05, T-1.ACCT.03
- **Acceptance Criteria**: Settlement of a foreign-currency document posts the realized difference to the configured gain/loss account and balances; revaluation at a chosen date posts the unrealized difference and is reversible/repeatable without double-counting; the gains/losses reconcile to the change in rate applied to the open balance.
- **Evidence**: `ERPbackend/app/ledger/fx_gain_loss.py` — both differences, both posted through the primitive so they cannot be unreported: `settle_document` (the difference between a document's value at the settlement rate and the base value its own postings carried, posted against its control account), `revalue_open_balance` (the same for an open balance at a chosen date's rate, **netting off what is already posted** so a second run of the same date posts nothing), `reverse_revaluation` (the negation, as a new entry — an append-only ledger is never edited) and `open_foreign_balance`/`already_revalued` as the reads. The gain and loss accounts are **configuration** (`fx_gain`/`fx_loss` mapping keys, T-1.ACCT.03), and the sign of the difference decides which one is used, so no caller has to say whether its account is an asset or a liability. Check: `ERPbackend/tests/check_fx_gain_loss.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_fx_gain_loss.py`, exit 0), green on all eight: a payable booked at 58.5 and settled at 59.75 posted a realized **loss of 125.000000** (credit on 2000, debit on 5990) equal to `100.00 × (59.75 − 58.5)`; settling at the booked rate posted nothing; an open receivable revalued `+125.000000` on 1100 credited to 4910, equal to `balance × Δrate`; the second run of the same date posted **nothing** (one revaluation entry in the ledger); the reversal cancelled it (net revaluation back to 0) and the revaluation could then be posted again; the base currency was refused a revaluation; and T-0.CORE.02's `ledger_gate` was green over the whole ledger. The full suite (19 checks) is green on the same tree.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: DONE

#### Task ID: T-1.ACCT.07
- **Title**: Basic financial reports — Trial Balance, P&L, Balance Sheet, Cash Flow
- **Description**: Produce the four statements from the GL, scoped by company and period, delivered through the reporting framework.
- **Plan source**: §2.1 Key Capabilities (Financial statements (Trial Balance, P&L, Balance Sheet, Cash Flow)), §4 Phase 1 bullet 5 (Basic financial reports)
- **Scope**: The four statements at basic depth. Advanced analytics and the broader scheduled catalogue are Phase 6.
- **Variables / Config**: `company_id`, `fiscal_year_start`, `period_lock_granularity`, `base_currency`, `report_schedule`.
- **Dependencies**: T-1.ACCT.03, T-0.REPORT.01
- **Acceptance Criteria**: Trial Balance debits equal credits for any selected period; P&L and Balance Sheet foot to the GL and the Balance Sheet balances (assets = liabilities + equity); Cash Flow is derivable from the entries; each report is scoped by company and permission and can be scheduled.
- **Evidence**: `ERPbackend/app/ledger/statements.py` — the four statements, all **derived from the entries** (no balance table, no snapshot): `trial_balance` (debit and credit totals per account, with the two totals and whether they agree), `profit_and_loss`, `balance_sheet` (with the period's earnings shown inside equity, which is what makes assets = liabilities + equity hold on any date while Phase 1 has no closing entries) and `cash_flow` (opening, the movements grouped by counterpart account, and a closing figure checked against the cash accounts' own balance). Every figure is restated at each entry's rate, so both currencies are stated in the company's base currency (T-1.ACCT.05). Each statement returns `balanced`, computed rather than asserted. The four are registered with T-0.REPORT.01's framework on import, so a definition (cron schedule + recipients, both rows) runs and delivers them; the API exposes `POST /api/v1/reports` and `POST /api/v1/reports/{code}/run`; contract republished (every documented field typed). Check: `ERPbackend/tests/check_financial_statements.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_financial_statements.py`, exit 0), green on all eight over a five-posting dataset (capital, a sale on account, rent, the customer settling, a cash sale): the trial balance's totals equal at `2150.000000` and equal the ledger's; the P&L reads income `650.000000`, expenses `200.000000`, net `450.000000`, cross-checked against the trial balance rows independently; the balance sheet balances (assets `1450.000000` = liabilities `0` + equity `1450.000000`, with Current Year Earnings `450.000000`); the cash flow's closing `1250.000000` equals `closing_per_ledger`, its movement rows sum to its net change, and its opening is the pre-period balance; the trial balance then ran through the framework (status `ok`, produced `balanced`) and was delivered to both recipients (two delivery rows written); and a caller without `report.read` was refused (`'bob.clerk' may not 'report.read' (holds clerk…)`). The full suite (19 checks) is green on the same tree.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: DONE

### Stream: `INV` — Inventory & Warehouse Management (§2.2)

#### Task ID: T-1.INV.01
- **Title**: Item master — SKU, variants, UOM conversion, barcode/QR
- **Description**: Model and expose the Item, Item Variant and UOM Conversion entities, with barcode/QR identifiers, and a base UOM with conversion factors to alternate UOMs.
- **Plan source**: §2.2 Core Components (Item Master (SKU, variants, UOM conversion, barcode/QR)), §4 Phase 1 bullet 3, §5 Item, Item Variant, UOM Conversion
- **Scope**: Item identity, attributes, variants, UOM conversion, identifiers. Valuation attributes (costing method) belong to T-1.INV.04; batch/serial attributes to T-1.INV.08 and T-1.INV.09.
- **Variables / Config**: `uom_conversion_factor`, `item_variant_attributes`, `barcode_symbology`, `traceability_mode`, `company_id`.
- **Dependencies**: T-0.MODELS.01, T-0.AUDIT.02
- **Acceptance Criteria**: A variant matrix is defined by attributes without creating duplicate SKUs; a barcode is unique system-wide and scanning it resolves to exactly one item/variant; UOM conversion is lossless for a round trip within its declared precision; conversion factors of zero are rejected.
- **Evidence**: `ERPbackend/app/stock/items.py` — `Item` (unique SKU per company, `base_uom`, `traceability_mode` with **no default**), `ItemVariant` (unique per attribute combination, unique SKU), `ItemBarcode` (value unique **across the installation**, `ean`/`upc`/`qr`), `UomConversion` (`factor > 0`, unique direction), plus `create_item`, `add_variant`, `add_barcode`, `item_from_barcode`, `add_uom_conversion`, `convert_quantity` and `rename_item` — the only field that may change after creation, because §5.1 makes `sku`, `base_uom` and `traceability_mode` immutable. Conversion walks the declared chain breadth-first and the reverse direction **divides by the same factor** rather than storing a rounded reciprocal, which is what makes a round trip exact where pre-rounding `1/12` would drift the quantity. Check: `ERPbackend/tests/check_item_master.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_item_master.py`, exit 0), green on all seven: an unknown traceability mode refused (`unknown traceability mode 'sometimes'`); two variants as one item with two SKUs and a repeated attribute combination refused; one item carrying an EAN and a QR with a scan resolving to item **and** variant; a duplicate barcode refused by the helper and by `uq_item_barcode_value` for another company's row; `2 pallets = 2400.000000 each` and back to `2.000000` exactly, `1 pallet = 100.000000 box`; `from = to`, a duplicate direction and a non-positive factor refused, with `ck_uom_conversion_positive` refusing a raw zero factor; an undeclared path refused (`'TEE' declares no conversion involving c…`); and a retired item's row kept with `DELETE` refused as `is a master`. The full suite (22 checks) is green on the same tree.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.02
- **Title**: Multi-warehouse location hierarchy (Warehouse → Zone → Aisle → Bin)
- **Description**: Model the hierarchical location tree and let every stock entry reference a leaf location, with tree integrity and per-location capacity/status at basic depth.
- **Plan source**: §2.2 Core Components (Multi-Warehouse Hierarchy (Warehouse → Zone → Aisle → Bin)), §4 Phase 1 bullet 3 (basic Warehouse hierarchy), §1 principle 4
- **Scope**: The hierarchy and location referencing. No bin-level capacity planning or put-away strategies (not named in the plan).
- **Variables / Config**: `warehouse_hierarchy_levels`, `company_id`.
- **Dependencies**: T-0.MODELS.01
- **Acceptance Criteria**: The four levels nest with the named types; a stock entry cannot reference a non-leaf location; a location with stock cannot be removed; the hierarchy is retrievable as a tree per company.
- **Evidence**: `ERPbackend/app/stock/locations.py` — `Location` (DOMAIN-MODELS.md §6 columns, unique code per company) with the four-level rule enforced **in the database** by a deferred constraint trigger: a zone's parent must be a warehouse, an aisle's a zone, a bin's an aisle, a warehouse takes none; a move into the location's own subtree and retiring a parent with live children are refused there as well; `deny_hard_delete` makes it a T-0.AUDIT.01 master. Helpers: `create_location`, `move_location`, `require_leaf` (the rule every stock movement asks), `retire_location` — refusing live children **and** a location holding stock, which it asks of T-1.INV.03's ledger rather than duplicating — and `location_tree`. Check: `ERPbackend/tests/check_location_hierarchy.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_location_hierarchy.py`, exit 0), green on all six: the four levels nest and read back as a tree (Warehouse → Zone → Aisle → Bin); a bin under a warehouse refused by the helper (`a bin cannot stand under a warehouse: the levels are wareh…`) **and** by raw SQL at COMMIT; a zone with no parent and a warehouse with a parent refused; a cycle refused at COMMIT (`location WH1 cannot be moved under…`); a parent with live children refused retirement by the helper and by a raw `UPDATE location SET deleted_at`; a duplicate code refused. The stock-side half of the same rules — a movement against a non-leaf refused, a location holding stock not removable — is proven in `tests/check_stock_ledger.py` (T-1.INV.03, same batch). The full suite (22 checks) is green on the same tree.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.03
- **Title**: Stock ledger entry model (quantity + value, append-only)
- **Description**: Persist every stock movement as an immutable ledger entry carrying quantity and value, the item, the location, and the source document.
- **Plan source**: §2.2 Core Components (Stock Ledger & Valuation), §4 Phase 1 bullet 3, §5 Stock Ledger Entry (qty + value), §1 principle 5
- **Scope**: The entry model and its read views. Valuation maths is T-1.INV.04.
- **Variables / Config**: `company_id`, `warehouse_hierarchy_levels` (via location), `transaction_currency` (value currency).
- **Dependencies**: T-1.INV.01, T-1.INV.02, T-0.CORE.01, T-0.AUDIT.01
- **Acceptance Criteria**: Every movement writes one entry with quantity and value; entries are append-only; the current quantity and value of an item at a location equal the sum of its entries; a movement without a source document is rejected.
- **Evidence**: `ERPbackend/app/stock/entries.py` — `StockLedgerEntry` exactly as DOMAIN-MODELS.md §7 fixes it (signed `quantity` and `value` in the item's base UOM, a leaf location, required `(source_type, source_id)`, the `batch_id`/`serial_id` columns whose presence is why traceability moved into Phase 1), `append_only` registered, and four table constraints: a movement moves something, its value keeps the quantity's direction, the source is a real pair, the currency is three letters. A **BEFORE INSERT trigger** enforces the leaf rule at the storage boundary, so a writer that bypasses the recorder cannot put stock in a warehouse. `record_movement` is the only way in (the quantity arrives signed by the transaction, in the base UOM), `on_hand` reads the balance as the ledger's own sum with optional item/location/variant/batch/date narrowing, `movements` feeds the valuation and `movements_for_source` is the drill-down. Check: `ERPbackend/tests/check_stock_ledger.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_stock_ledger.py`, exit 0), green on all eight: three movements summing to `80.000000` on hand and `410.000000` in value, narrowed per location (`70` / `10`) and by date (`100` on the first day); `UPDATE`/`DELETE` on a stored row refused as `append-only`; a movement with a blank source refused by the recorder (`a stock movement names the document that cause…`) and by `ck_stock_entry_source`; a movement against a warehouse refused (`WH1 is a warehouse; a stock movement references a bi…`) and by the trigger (`stock is held in a bin`); a zero movement refused by the helper and by the table; a value fighting the direction refused; a bin holding stock refused retirement (`B1 still holds 70.000000`); and one document's movements found by its id. The full suite (22 checks) is green on the same tree.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.04
- **Title**: Valuation engine — FIFO / Moving Average / Standard Cost with real-time valuation
- **Description**: Compute stock value per the configured costing method and expose real-time valuation for any item and location, including standard-cost revaluation handling.
- **Plan source**: §2.2 Core Components (Stock Ledger & Valuation (FIFO, Moving Average, Standard Cost)), §2.2 Key Capabilities (Real-time stock valuation), §4 Phase 1 bullet 3/4
- **Scope**: The costing methods, their selection and real-time valuation. Physical counts and adjustments are T-1.INV.06; GL posting is T-1.INV.07.
- **Variables / Config**: `costing_method`, `valuation_scope`.
- **Dependencies**: T-1.INV.03
- **Acceptance Criteria**: Each of the three methods produces the method-correct value for the same movement sequence; the method is selectable at the configured scope and changing it does not rewrite history; valuation is available without a batch job; a method that cannot value an item (e.g. standard cost with no standard set) fails loudly rather than valuing at zero.
- **Evidence**: `ERPbackend/app/company.py` gained `costing_method` (the three §2.2 names, default `moving_average`, resolved **per company** — the decided `valuation_scope`), and `ERPbackend/app/stock/valuation.py` the engine: `valuation()` walks the ledger in posting order per method (moving average against the running average cost, FIFO consuming the oldest layers, standard cost as `quantity × standard cost`), `value_issue()` prices an issue from the same walk — the difference a synthetic issue makes — and `set_costing_method()` refuses a method outside the three. Nothing is stored and nothing is scheduled: the valuation is computed on read, which is why changing the method rewrites no history. Check: `ERPbackend/tests/check_stock_valuation.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_stock_valuation.py`, exit 0), green on all seven over one movement sequence (receive 100 @ 5.00, receive 50 @ 6.00, issue 120, receive 30 @ 7.00): the same 60 units value at **moving average 370.000000**, **FIFO 390.000000** and **standard cost 360.000000**; the engine prices that one issue at 640.000000 / 620.000000 / 720.000000 respectively, and the issue was recorded at the engine's own figure; Standard Cost without a standard is refused (`'NUT' has no standard cost, so Stan…`); switching the method left the ledger rows identical (count and value sum compared before and after); an unknown method is refused; and valuation narrows by location (`30.000000` at B2) and by date (`150.000000` as of 2026-09-05). The full suite was green on the same tree.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: DONE

#### Task ID: T-1.INV.05
- **Title**: Stock transactions — Receipt, Issue, Transfer
- **Description**: Implement the three Phase 1 stock transactions, each writing stock ledger entries and honouring the location hierarchy and UOM conversion.
- **Plan source**: §2.2 Core Components (Stock Transactions (Receipt, Issue, Transfer, Reconciliation)), §4 Phase 1 bullet 4
- **Scope**: These three transaction types only. Reconciliation/adjustment is split out as T-1.INV.06 (the plan names them in §2.2 and schedules none, so they are placed with the transaction family in Phase 1 — see Open Questions).
- **Variables / Config**: `costing_method` (via valuation), `uom_conversion_factor`, `warehouse_hierarchy_levels`.
- **Dependencies**: T-1.INV.04
- **Acceptance Criteria**: Receipt increases quantity and value at the target location; issue decreases it and cannot drive quantity negative unless negative stock is explicitly permitted; transfer moves quantity and value between locations with no net change in total quantity/value; entries in a foreign UOM convert to the base UOM correctly; each transaction is atomic.
- **Evidence**: `ERPbackend/app/stock/transactions.py` — `receive`, `issue` and `transfer`, each converting the caller's UOM to the item's base UOM (T-1.INV.01), refusing a non-positive quantity and writing through T-1.INV.03's recorder; a receipt states its total value, an issue is priced by T-1.INV.04's engine, a transfer carries the value across both of its entries; an issue or transfer beyond what a location holds is refused (`allow_negative_stock` is `false`). Nothing commits — the caller's document owns the transaction, so a failure leaves no movement. Check: `ERPbackend/tests/check_stock_transactions.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_stock_transactions.py`, exit 0), green on all seven: `2 boxes arrived as 24 each, worth 240.00` (UOM conversion at the edge); an issue of 4 valued at `-40.000000` by the engine; an issue beyond the location refused (`B1 holds 20.000000 of 'BOLT'; issuing 100.000000 would t…`) with **no** movement left behind; a transfer of 10 moving `100.000000` with the goods and leaving the item's totals unchanged; a transfer out of a short location, and a transfer to the location it came from, both refused; all four movements naming their document; and the ledger-integrity gate green over them.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.06
- **Title**: Physical count and reconciliation/adjustment workflow
- **Description**: Support counting a location (or a subset of items), recording counted quantities, and posting the resulting adjustment with approval and an audit trail.
- **Plan source**: §2.2 Core Components (Reconciliation), §2.2 Key Capabilities (Physical count and adjustment workflows), §1 principle 6
- **Scope**: Count sheets, variance calculation, adjustment posting. No cycle-count scheduling automation (not named in the plan).
- **Variables / Config**: `approval_levels`/`approval_thresholds` (adjustment approval), `costing_method` (adjustment value).
- **Dependencies**: T-1.INV.05, T-0.WF.01
- **Acceptance Criteria**: A count produces a variance per item against system quantity; posting an adjustment creates ledger entries for exactly the variance; an adjustment above the configured threshold requires approval before it affects the ledger; the count, the approver and the adjustment are all auditable.
- **Evidence**: `ERPbackend/app/stock/counts.py` — `PhysicalCount`/`PhysicalCountLine` (the system quantity snapshotted when the count opens, the counted quantity and the variance), `start_count`, `record_count`, `adjustment_value` and `post_adjustment`. The adjustment's value goes through T-0.WF.01's engine as document type `inventory_adjustment`: when the configured chain routes it, posting without an **approved** request is refused before anything is written, and a count already posted refuses to post again. The count's rows are company-scoped (so the audit trigger records them) and the entries name the count as their source. Check: `ERPbackend/tests/check_physical_count.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_physical_count.py`, exit 0), green on all six: the count snapshots 100 bolt and 50 nut; counting 99 and 51 gives variances `-1.000000` and `+1.000000` and an adjustment worth `30.000000` — below the configured 50 threshold, so no approval; the adjustment posted **exactly** those two variances and the bin then held 99; the entries name the count and the count is on the trail; an adjustment of `90.000000` was refused while its request was `pending` (`the adjustment of 90.000000 ne…`) with no movement written; and after an approver at the configured role approved it, the same call posted `+9.000000` once and posting again was refused. Carried with it: `physical_count_line` is declared in `app/db.py`'s `CHILD_TABLES`, which the scoping guard demanded — the convention working as intended.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.07
- **Title**: Post stock movements to the GL inventory account and reconcile valuation to GL
- **Description**: Wire stock ledger activity into the GL through T-1.ACCT.03 so the inventory account reflects stock value, and provide the reconciliation that proves the two agree (metric 2 of §6).
- **Plan source**: §2.2 Key Capabilities (Real-time stock valuation), §3 row "Double-Entry Integrity", §6 metric 2, §4 Phase 1 exit criteria
- **Scope**: The posting wiring and the reconciliation check. Not the valuation maths (T-1.INV.04) nor the statements (T-1.ACCT.07).
- **Variables / Config**: account mapping (inventory, stock adjustment, COGS), `costing_method`.
- **Dependencies**: T-1.INV.05, T-1.ACCT.03
- **Acceptance Criteria**: Every receipt, issue, transfer and adjustment posts a balanced entry to the chart of accounts; the GL inventory balance equals the sum of stock ledger values for the same period to the unit of currency precision; a deliberate mismatch is detected and reported, not silently accepted; reconciliation can be run per company and period.
- **Evidence**: `ERPbackend/app/stock/gl_posting.py` — `post_movement_to_gl` (the inventory account takes the movement's value with its sign, the configured counterpart takes the opposite, so every entry balances by construction), `account_key_for` with a **refusal** for a source type nobody mapped rather than a guessed account, and `reconcile` (the GL inventory account's movement in base currency against the stock ledger's own value movement, with `balanced` and the difference as figures). The three transactions of T-1.INV.05 and the adjustment of T-1.INV.06 call it immediately after they write their entry, in the same transaction, so a movement without its posting is not a reachable state. Check: `ERPbackend/tests/check_stock_to_gl.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_stock_to_gl.py`, exit 0), green on all six: a receipt posting `Dr 1200 1000.000000 / Cr 2000 1000.000000` and an issue the mirror image (`Cr 1200 100.000000 / Dr 5000 100.000000`); a transfer posting both halves against 1200 alone (traceable, total unmoved); an adjustment posting its variance; `stock 850.000000 = GL 850.000000; difference 0` — §6 metric 2 — with the GL account read back independently; a movement whose posting was skipped reported as `difference -100.000000` and cleared once posted; and an unmapped source type refused (`no account mapping is known for 'stock_count…`) with the integrity gate green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.08
- **Title**: Batch/Lot tracking with expiry
- **Description**: Extend the stock ledger entry and the stock transactions with batch/lot identity, expiry dates and a FEFO issue suggestion, for items whose `traceability_mode` is Batch/Lot.
- **Plan source**: §2.2 Core Components (Traceability (Batch/Lot with expiry, Serial numbers)), §5 Batch / Serial — **scheduled in Phase 1 by the 2026-09-17 revision** (was §4 Phase 6)
- **Scope**: Batch identity across the Phase 1 stock transactions and the Phase 2 receipts that reuse them. Serial tracking is T-1.INV.09. Extends the entry model of T-1.INV.03 rather than replacing it.
- **Variables / Config**: `traceability_mode` (Batch/Lot), `batch_expiry_required`, `costing_method` (batch-level valuation).
- **Dependencies**: T-1.INV.05, T-1.INV.03
- **Acceptance Criteria**: Batch identity is required on every movement of a batch-tracked item and a movement without it is refused; items whose mode is `None` are unaffected (no regression to T-1.INV.05); batch quantities reconcile to the item's total quantity; an expired batch cannot be issued without an explicit, audited override; issue suggestion follows FEFO; valuation remains correct per batch under the item's costing method.
- **Evidence**: `ERPbackend/app/stock/batches.py` — `Batch` (code unique per item, nullable `expiry_date` because `batch_expiry_required` is "Not stated", retired by marking), `create_batch` (which refuses a batch on an item not tracked by batch), `require_usable` (an expired batch refused, and an override that must **name the actor** who took the decision — the origin is then on the audit trail of the movement that follows), `fefo_batch` (earliest expiry holding stock at a location) and `retire_batch` (refused while the batch holds stock). The enforcement itself is in T-1.INV.03's recorder: a `batch_lot` item moves only with its batch, a `none` item never with one, and `stock_ledger_entry.batch_id` is a foreign key to `batch`. Check: `ERPbackend/tests/check_batch_tracking.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_batch_tracking.py`, exit 0), green on all seven: a batch-tracked movement with no batch refused (`'MILK' is tracked by batch…`) and an untracked item carrying one refused; receipts and an issue by batch leaving `MILK-SOON` holding 25 with the item totalling 45; FEFO suggesting `MILK-SOON`, the soonest expiry with stock; an expired batch refused on issue; the same issue allowed with `allow_expired=True` **and** the actor named, with the override's origin on the trail; per-batch valuation reading that batch's own value (`500.000000` for 25 units); and an untracked item moving, transferring and valuing as before.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-1.INV.09
- **Title**: Serial number tracking
- **Description**: Track individually serialised units through receipt, issue and transfer, with a status per serial, for items whose `traceability_mode` is Serial. Sales and returns exercise it from Phase 3.
- **Plan source**: §2.2 Core Components (Traceability (… Serial numbers)), §5 Batch / Serial — **scheduled in Phase 1 by the 2026-09-17 revision** (was §4 Phase 6)
- **Scope**: Serial identity lifecycle and status within the Phase 1 stock transactions. Service/warranty features are not named in the plan.
- **Variables / Config**: `traceability_mode` (Serial), `allow_negative_stock` (never).
- **Dependencies**: T-1.INV.05, T-1.INV.03
- **Acceptance Criteria**: A serial-tracked item's quantity always equals its count of distinct serials in stock; a serial can only be in one place at one time; receiving, transferring and issuing a serial transitions its status correctly; a duplicate serial within an item is refused while the same value under a different item is allowed.
- **Evidence**: `ERPbackend/app/stock/serials.py` — `Serial` (code unique **per item**, so the same value on another item is a different unit; status `in_stock`/`issued`; the location while it is in stock), `add_serial`, `serial_by_code`, `place_serial` (receiving puts the unit in the location, issuing takes it out) and `serials_in_stock`. T-1.INV.03's recorder enforces the identity: a `serial` item moves one named unit at a time (a movement of more than one, or of none, is refused), and `stock_ledger_entry.serial_id` is a foreign key to `serial`. Check: `ERPbackend/tests/check_serial_tracking.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_serial_tracking.py`, exit 0), green on all six: a movement with no serial refused (`'PUMP' is tracked by seria…`) and a three-unit movement refused; SN-0001 received at B1, transferred to B2 and issued out of stock, its status and location following; the item's quantity (`1`) equal to its count of distinct serials in stock, naming `SN-0002`; a duplicate serial within the item refused while the same code on another item was allowed; issuing a unit that is already out refused; and an untracked item moving exactly as before.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: DONE

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
- **Evidence**: `ERPbackend/tests/check_phase1_exit.py`, run 2026-09-19 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_phase1_exit.py`, exit 0), builds one dataset covering every Phase 1 kind of work (receipt, issue, transfer, count/adjustment, batch receipt and issue, serial receipt, a manual journal entry, a USD invoice settled, a USD receivable revalued) and verifies each criterion of this gate **in the tree**: (1) *posting* — the writer scan finds `app/ledger/posting.py` as the only writer of the two ledger tables while four modules posted through it, and an unbalanced call is refused (`entry does not balance: debit 1 !=…`) — §6 metric 1; (2) the ledger-integrity gate reports **zero imbalances** over the whole dataset; (3) *valuation* — after receipts, an issue, a transfer and an adjustment, 85 bolt value at moving average `850.000000`, FIFO `850.000000` and standard cost `510.000000`, with `value_issue` of 5 units at `50.000000`; (4) *reconciliation* — `valuation 2050.000000 = GL 1200; difference 0.000000`, the GL account read back independently — §6 metric 2; (5) *period locking* — with 2026-09 closed a receipt dated inside it is refused (`2026-09 is closed for posting; unlock the pe…`) and the ledger is unchanged; (6) *multi-currency* — the USD document posted at its dated rate, the settlement posted a balanced realized loss of `125.000000` on 5990 and the revaluation a balanced unrealized gain of `125.000000` on 4910 (debit-positive, so the credited gain reads `-125.000000` there); (7) *statements* — trial balance `20200.000000 = 20200.000000` and balance sheet `13025.000000 = 8325.000000 + 4700.000000`, both `balanced`; (8) *traceability* — a batch-tracked item moved without a batch, and a serial-tracked one without its unit, are both refused while untracked items keep their quantities.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: DONE

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
- **Evidence**: `ERPbackend/app/procurement/suppliers.py` (new package `app/procurement/`) — the `Supplier` profile keyed one-to-one to T-0.PARTY.01's `Party` (`uq_supplier_company_party`), carrying `payment_terms_days` (**no default**: 0 is a real answer meaning due on receipt) and an optional `transaction_currency` resolved through T-1.ACCT.05's master (`None` = the company's base currency, never a copied base), plus three detail tables that each carry the company dimension (`supplier_contact`, `supplier_bank_account`, `supplier_tax_identifier`) with a **partial unique index** (`postgresql_where=text("is_primary")`) for the one primary contact and the one primary bank account. `create_supplier` gives a party that already exists the supplier role instead of founding a second record, so §5's one-party-many-roles rule holds; e-mail, SWIFT and account-number shapes are validated where they are written; and `documents_naming` **scans the schema's foreign keys** to `supplier`, so retirement starts being refused the day a document table appears rather than when someone remembers to register it. Check: `ERPbackend/tests/check_supplier_master.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_supplier_master.py`, exit 0) — green on all eight: a supplier party created with the supplier role; one party holding customer **and** supplier with one party row and one tax number; a second profile for the same party refused; a malformed e-mail, `not-a-number` account, a 3-character SWIFT, a blank tax value, an unknown identifier kind and an unregistered currency all refused **at entry**; a well-formed contact/bank/TIN stored with the primary lookups resolving; a new primary demoting the previous one while the database's partial index refuses a hand-written second; retirement refused by name while a `probe_purchase_order` row names the supplier (`SupplierInUseError`) and a free supplier marked instead — its row still present and `DELETE` refused (`is a master: DELETE is refused`); a retired supplier no longer resolving by code, a non-supplier party refused; and the same party code existing in two companies, each resolving inside its own.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.02
- **Title**: Purchase requisitions with approval workflow
- **Description**: Let requesters raise purchase requisitions and route them through the configured multi-level approval chain before they can progress.
- **Plan source**: §2.3 Core Components (Purchase Requisitions + approval workflows), §4 Phase 2 bullet 1, §3 row "Workflows & Approvals"
- **Scope**: Requisition document, lines, and approval routing. Sourcing happens in RFQ (T-2.PROC.03).
- **Variables / Config**: `approval_levels`, `approval_thresholds`, `company_id`.
- **Dependencies**: T-2.PROC.01, T-0.WF.01
- **Acceptance Criteria**: A requisition above the configured threshold cannot be sourced until every level approves; rejection closes or returns it per the workflow; an approved requisition is immutable except by a new revision; approval history is complete and auditable.
- **Evidence**: `ERPbackend/app/procurement/requisitions.py` — `Requisition` + `RequisitionLine` (company-scoped, `quantity > 0`, `estimated_unit_price >= 0`, one line per `line_no`, and a check that makes `revision_no = 1` and `revision_of_id` agree). The approval half is **T-0.WF.01's**, not a copy: `submit` hands the document to the shared engine under `doc_type = "purchase_requisition"` and mirrors the engine's resulting state, `record_decision` delegates to `workflow.decide`, and `revise` raises a **new draft** carrying the original's lines while the original keeps its state and its history. `requisition_total` is derived from the lines every read, so a stored total cannot disagree with them, and `require_sourceable` is the gate T-2.PROC.03/T-2.PROC.05 call. Check: `ERPbackend/tests/check_requisition_approval.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_requisition_approval.py`, exit 0) — green on all eight: an exact total (25 × 1000.00 + 3 × 0.005 = 25000.015000); a repeated number, a blank description, a zero quantity, a negative price, an unregistered currency and an empty submission all refused; a 25000.015 requisition routed to a **two-level** chain and refused sourcing both before level 1 and **between** level 1 and level 2; a decision from the wrong role refused (`level 1 of purchase_requisition is decided by 'manager', not 'director'`), a reasonless rejection refused, a rejection closing the document and a return leaving it `returned`; a requisition below every threshold approved **on submission** with no approval request at all, and a company with no chain refused rather than waved through; an approved requisition refusing a new line while `REQ-001-R2` was raised as a draft over the same lines with the original still `approved`; the history read back in order from the engine's append-only decisions; and a draft refusing revision plus a second revision refused **by the database** (`uq_purchase_requisition_revision_of_id`).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.03
- **Title**: RFQ creation and supplier response capture
- **Description**: Raise a request for quotation against approved requisition lines, invite suppliers, and record their quoted prices, lead times and validity.
- **Plan source**: §2.3 Core Components (Request for Quotation (RFQ) & Supplier Portal)
- **Scope**: RFQ issue and response capture in the application. The supplier-facing portal is Phase 6 (T-6.PORTAL.01).
- **Variables / Config**: `tax_pack` (quote tax), `transaction_currency`, `approval_levels` (if an RFQ needs approval — not stated in the plan).
- **Dependencies**: T-2.PROC.02
- **Acceptance Criteria**: An RFQ can be issued to multiple suppliers with a response deadline; responses are recorded per line per supplier; a late response is recorded as late rather than silently accepted; the RFQ shows which requisition lines each response covers.
- **Evidence**: `ERPbackend/app/procurement/rfq.py` — `Rfq` (issued against an **approved** requisition, `require_sourceable` called before anything else), `RfqLine` carrying `requisition_line_id` so coverage is exact rather than matched by description, `RfqSupplier` (invited **whether or not** they answer), `RfqResponse` (received_on, `late`, currency, lead time, validity) and `RfqResponseLine` (`unit_price >= 0`, one row per line per response). `record_response` refuses a second answer from the same supplier, refuses a line the RFQ never asked about, and stores a post-deadline answer with `late = true` instead of accepting or dropping it silently; `non_responders` returns the invited suppliers with no response row, which is what keeps "did not answer" distinct from "answered zero". Check: `ERPbackend/tests/check_rfq_responses.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_rfq_responses.py`, exit 0) — green on all eight: an RFQ issued to ACME and BOREAL over REQ-100's two lines with each RFQ line keeping its requisition line; issuing refused while REQ-100 was still `pending` (`it cannot be sourced until every level…`); ACME's two-line answer with its 21-day lead time and validity captured and both requisition lines covered; line 9 refused (`it asks about [1, 2]`), a negative price refused and an empty answer refused; BOREAL's 2026-10-14 answer stored `late = true` against a 2026-10-10 deadline while ACME's stayed `false`; CHIRP refused as never invited and a second ACME answer refused; ACME quoting **0** for a line while CHIRP is a non-responder — the two told apart; and an RFQ to nobody, a deadline before issue, a repeated number and a party that is not a supplier all refused, with a closed RFQ taking no further answers. **The API half** (published here so the API-first ordering holds and T-2.PROC.04 has a contract to build against): `POST /api/v1/rfqs`, `POST /api/v1/rfqs/{number}/responses` and `GET /api/v1/rfqs/{number}` in `app/api.py`, with `contract/v1/openapi.json` republished (`tools/publish_contract.py`, +755 lines) and the frontend's copy re-vendored byte for byte. The read endpoint carries the facts the matrix needs and no more: each quoted unit price with the `fx_rate` it was converted at and the resulting `base_unit_price`, and an `RfqBasisOut` naming the base currency, the rate date, the pack's procurement tax rule and its rate — so the client can *label* the basis instead of inferring it. Verified in the same check: an API round-trip asserts the basis payload (`VAT-IN-12`, 12 %, `fx_on = 2026-10-01`), ACME's PHP quote unconverted, BOREAL's USD 17.000000/1005.000000 quote converted at 58.5000000000 to 58792.500000, the late flag riding with the number, and an actor holding no capability refused with 403. `check_api_conventions.py` re-run green (the served contract equals the published artifact; 137 documented fields typed and nullable where optional).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.04
- **Title**: Comparative statement matrix
- **Description**: Present supplier responses side by side per line with price, tax, lead time and landed comparison, so an award can be justified.
- **Plan source**: §2.3 Key Capabilities (Comparative statement matrix)
- **Scope**: Comparison view and its export. Awarding is T-2.PROC.05.
- **Variables / Config**: `tax_pack`, `transaction_currency`.
- **Dependencies**: T-2.PROC.03
- **Acceptance Criteria**: Every responding supplier appears per line with its quoted values; the comparison uses a consistent basis (converted to base currency and tax-inclusive as configured) and labels the basis; a supplier that did not respond is distinguishable from one that quoted zero; the matrix can be exported.
- **Evidence**: `ERPfrontend/lib/comparison.ts` (frontend repository, where the `## Repositories` map places this task) — `buildComparison` turns one `RfqOut` (the contract type, so the matrix cannot drift from the published API) into rows of `ComparisonCell`s and `comparisonCsv` writes the same thing out in **long** form (one row per supplier per line) with the basis repeated on every row. The basis itself is computed **exactly**, never through `Number`: amounts are decimal strings scaled to 10^-6 integers, so `taxInclusive` and `multiply` round half-up at the money scale and a cent can never be invented by binary floating point. Three states stay distinct — `quoted`, `no_quote` (answered, silent about that line) and `did_not_respond` — which is what stops a silent supplier reading as a free one. The screen `app/rfqs/[number]/page.tsx` renders the matrix with the basis printed above it and the late answers flagged; `app/rfqs/[number]/export/route.ts` serves the CSV at a URL. `lib/api.ts` gained `readRfq` and `instanceIdentity`, and the vendored `contract/v1/openapi.json` + `lib/contract.d.ts` were regenerated from the backend's republished artifact (byte-identical — `diff -q` clean). Check: the frontend pipeline's own `npm run check` (mirroring `.github/workflows/frontend.yml`), run 2026-09-29 — green on all six steps (no database driver or backend source in any of the six scanned files; the vendored contract byte-identical to the backend's artifact; the generated types current; a clean typecheck; a field removed from the contract breaking the build; the Next.js production build succeeding) **plus** `tests/comparison.test.ts`, now run by that check: 6 tests / 10 subtests green — every responding supplier per line with its quoted, converted and tax-inclusive values and an exact extended total; the basis label on the matrix *and* on all six CSV rows; did-not-respond vs quoted-zero vs no-quote told apart; a late answer compared and flagged; the arithmetic exact on the cases a float misrounds (`0.1 × 3`, `99999999999.999999 × 1.12`, a half-up at the sixth decimal); and the CSV usable — one row per supplier per line and a comma-bearing description quoted (`"Laptops, 14 inch"`).
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.05
- **Title**: Automated PO generation from RFQ award
- **Description**: Award one or more RFQ lines to suppliers and generate the purchase order(s) automatically from the winning responses.
- **Plan source**: §2.3 Key Capabilities (Automated PO generation from RFQ)
- **Scope**: Award and PO generation. The PO's own lifecycle is T-2.PROC.06.
- **Variables / Config**: `transaction_currency`, `tax_pack`, `approval_levels` (if PO approval applies).
- **Dependencies**: T-2.PROC.03, T-2.PROC.04
- **Acceptance Criteria**: Awarding generates a PO whose lines carry the awarded price, supplier, requisition reference and required date with no re-keying; an award cannot exceed the requisitioned quantity without explicit override; a partial award leaves the remaining lines available for award.
- **Evidence**: `ERPbackend/app/procurement/orders.py` — `PurchaseOrder`/`PurchaseOrderLine` (the order keeps `requisition_id` **and** `rfq_id`; each line keeps its `rfq_line_id` and `requisition_line_id`, which is what makes T-2.MATCH.01's comparison answerable later) and `award()`, which writes the order from the winning **response's own lines** so no price is re-keyed. `remaining_awardable` derives what is still open per RFQ line from the orders already raised rather than from a counter that could drift, and an over-award refuses unless `override_reason` is given, which is then stored on the order. Every entry is validated **before** the order row is created, so a refused award leaves nothing half-written behind. Check: `ERPbackend/tests/check_po_generation.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_po_generation.py`, exit 0) — green on all seven: PO-2001 generated from ACME's quote carrying 980.00/255.50 with the requisition reference and the required date and totalling exactly 21980.000000; a partial award (12 of 20, all 40) leaving `{1: 8, 2: 0}` open and BOREAL then awarded the remaining 8 **at its own price** (1005.00); a non-responder (CHIRP) and a line a responder was silent about (BOREAL's line 2) both refused with no price invented; over-awarding refused (`asked for 20.000000 and 20.000000 is already awarded`) and accepted only with a reason recorded verbatim on the order; an award against an unapproved requisition refused; and a repeated order number, a zero quantity, an empty award and an unstated actor all refused, with `order_by_number` resolving the generated order.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.06
- **Title**: Purchase order lifecycle — approve, amend, close
- **Description**: Manage the PO through approval, amendment with revision history, partial fulfilment and closure.
- **Plan source**: §2.3 Core Components (Purchase Orders), §4 Phase 2 bullet 1
- **Scope**: PO states and transitions. Receiving is T-2.PROC.07.
- **Variables / Config**: `approval_levels`, `approval_thresholds`.
- **Dependencies**: T-2.PROC.02, T-2.PROC.05
- **Acceptance Criteria**: A PO above threshold requires approval before it can be sent or received against; an amendment after approval produces a new revision without erasing the approved one; a PO cannot close with open receipts unless an authorized short-close workflow requires a reason that is recorded on the order; the status transitions are recorded with actor and timestamp.
- **Evidence**: `ERPbackend/app/procurement/orders.py` (lifecycle added to the module that owns the tables) — `submit_order`/`decide_order` route the order through **T-0.WF.01** under `doc_type = "purchase_order"` and mirror the engine's state, `require_approved` is the gate a receipt calls, `amend` raises a **new draft revision** carrying the original's lines (refusing to amend a draft, a second revision, a negative price or a quantity below what has already arrived), `receipt_progress` derives ordered/received/outstanding per line, and `close_order` refuses while anything is outstanding unless a short-close reason is stated and recorded on the order. The `PurchaseOrderLine` gained `received_quantity` (moved only by a **posted** receipt, T-2.PROC.07) and the order gained `close_reason`; `status_trail` reads actor and time from **T-0.AUDIT.02's trail** rather than from a second table of its own. Check: `ERPbackend/tests/check_po_lifecycle.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_po_lifecycle.py`, exit 0) — green on all seven: PO-3001 (20000.000000) routed to a two-level chain and refused receiving while `pending` (`it cannot be sent or received against until…`) then approved after both levels; a director deciding the manager's level refused (`level 1 of purchase_order is decided by 'manager'…`), a reasonless return refused and PO-3002 left `returned` with its reason; PO-3003 (50.000000, below every level) approved on submission with **no** approval request; PO-3001-R2 raised as a draft over the same lines with the quantity changed to 5 while PO-3001 stayed `approved` with its original 4 and the chain read back `['PO-3001', 'PO-3001-R2']`; amending a draft and raising a second revision both refused; closing with 4.000000 outstanding refused and then closed short with the reason recorded verbatim; and a draft refusing closure with PO-3003's trail showing its actor and time for every transition.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.07
- **Title**: Goods Receipt Note (GRN) against a purchase order
- **Description**: Receive goods against PO lines into a warehouse location, creating stock ledger entries and recording rejected/partial receipts.
- **Plan source**: §2.3 Core Components (Goods Receipt Notes (GRN)), §4 Phase 2 bullet 2
- **Scope**: Receipt document and its stock effect. Invoice matching is T-2.MATCH.01.
- **Variables / Config**: `warehouse_hierarchy_levels`, `uom_conversion_factor`, `costing_method` (receipt value).
- **Dependencies**: T-2.PROC.06, T-1.INV.05
- **Acceptance Criteria**: A GRN increases stock at the chosen location and links to its PO lines; over-receipt beyond the PO quantity (plus any tolerance) is refused or explicitly overridden; a partial receipt leaves the remaining quantity receivable; the receipt's value flows into stock valuation.
- **Evidence**: `ERPbackend/app/procurement/receipts.py` + `app/procurement/orders.py` — `GoodsReceipt`/`GoodsReceiptLine` against an **approved** order (`require_approved`), each line carrying accepted and **rejected** quantities, the ordered unit price (never re-keyed) and the `movement_id` of the stock entry that posting produced. `post_receipt` calls :func:`app.stock.transactions.receive`, so the stock ledger entry and its balanced GL posting are written in the caller's transaction, and moves `PurchaseOrderLine.received_quantity` **only on posting** — which is what makes T-2.PROC.06's "a PO cannot close with open receipts" fall out of one rule instead of two. The item now flows requisition line → RFQ line → order line (`RfqLine.requisition_line`, `PurchaseOrderLine.item`) so a receipt has something to move. Check: `ERPbackend/tests/check_goods_receipt.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_goods_receipt.py`, exit 0) — green on all eight: a full chain requisition → RFQ → award → approval → receipt with 4 of 10 arriving at `MAIN-B1` worth 400.000000, linked to PO line 1 through its own movement and with a balanced GL entry; `received_quantity` moving to 4.000000 and 6.000000 left receivable; receiving against a **draft** order refused; over-receipt refused at posting (`asked for 10.000000 and 10.000000 has already arrived`) and accepted only with a reason recorded on the receipt; a rejection of 3 kept on the document while on-hand stayed 10; a draft receipt leaving the quantity outstanding so `close_order` refused; a posted receipt refusing a second posting; and an unknown order line, an empty receipt, a repeated number and a location from another company all refused.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.08
- **Title**: Supplier tax compliance and tax rule application
- **Description**: Apply the localization pack's tax rules to procurement documents and validate supplier tax identifiers, producing tax-inclusive values that downstream AP postings reuse.
- **Plan source**: §2.3 Key Capabilities (tax compliance), §3 row "Localization"
- **Scope**: Tax rule application on procurement documents and supplier tax data. Statutory filing outputs are not named in the plan and are not built here.
- **Variables / Config**: `tax_pack`, `company_id`.
- **Dependencies**: T-2.PROC.01, T-0.LOC.01
- **Acceptance Criteria**: Tax is computed per the active pack for each document type; a supplier with an invalid or missing tax identifier is flagged before the document is approved; tax amounts on the PO, GRN and invoice share one basis so the later match compares like with like; no market is hard-coded.
- **Evidence**: `ERPbackend/app/procurement/tax.py` — `active_market()` resolves the market from the packs the deployment ships (**no market name appears in the module**), `rules_for` reads the pack's own rules for a document type and refuses when the pack states none, `procurement_rule()` pins **one** rule for the whole chain (`purchase_order`, `goods_receipt`, `supplier_invoice`) and refuses when a pack taxes them differently because T-2.MATCH.01 would then compare unlike things, `tax_on` returns the rule code, rate, exact tax and total via T-0.LOC.01's `amount_from` (one rounding rule for the platform), and `findings`/`require_supplier_tax` check a supplier's tax data **before a document is approved** — a non-zero rate with no TIN behind it is a finding, a zero-rated one is not. Check: `ERPbackend/tests/check_supplier_tax.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_supplier_tax.py`, exit 0) — green on all seven: the module's source **scanned** and containing no installed market name while the market resolves to `'philippines'`; `VAT-IN-12` at 12 % on 2500.00 giving 300.000000 and 2800.000000; all three procurement documents returning that same code and the same 300.000000 on the same basis; `payroll_run` refused (`the pack states no tax rule for 'payroll_run'`); a supplier with no TIN flagged and refused while one holding `tin`/`vat` passes and its identifiers read back case-insensitively; exactness at money scale incl. a zero basis; and the rules read from pack `'philippines'` v1.0.0.
- **Estimated Effort**: M
- **Owner Role**: Domain Analyst (Accounting) + Backend Engineer
- **Status**: DONE

#### Task ID: T-2.PROC.09
- **Title**: Supplier performance scoring and vendor scorecards
- **Description**: Score suppliers from actuals — on-time delivery, receipt-to-PO quantity variance, quality/rejection rate and price variance — and present a scorecard.
- **Plan source**: §2.3 Key Capabilities (Supplier performance scoring, Vendor scorecards), §4 Phase 2 bullet 4 (performance basics)
- **Scope**: Basic scoring from the data produced in Phase 2. Cross-period trend analytics belong to Phase 6.
- **Variables / Config**: scoring weights (per company — not stated in the plan), `company_id`.
- **Dependencies**: T-2.PROC.06, T-2.PROC.07
- **Acceptance Criteria**: Each score derives from recorded documents, not manual entry; the inputs (promised date vs receipt date, ordered vs received quantity, rejections) are traceable to their source documents; the scorecard shows the weights used; a supplier with no receipts is shown as unrated rather than scored zero.
- **Evidence**: `ERPbackend/app/procurement/scoring.py` — `scorecard()` derives four figures from **posted** receipts only: on-time (receipt date against the order's required date), quantity (received against ordered), quality (accepted against accepted+rejected, both already on the receipt line) and price (awarded against the requisition's estimated price — the variance Phase 2 can evidence from its own documents; invoice-price comparison is T-2.MATCH.01's three-way job). Every figure carries an `inputs` row naming the order, receipt and line it came from, the weights are returned with the score (`scoring_weights`, equal by default because the plan leaves `scoring_weights` undecided), and a supplier with no receipts is `rated: False` with **no score** rather than zero. Check: `ERPbackend/tests/check_supplier_scorecard.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_supplier_scorecard.py`, exit 0) — green on all seven: GOOD scoring 100.000000 from `PO-501`/`GRN-501`; the unposted `GRN-503` absent from the scorecard entirely; POOR scoring 48.750000 from `{on_time 0, quantity 60, quality 60, price 75}` and ranking below GOOD; NEW unrated with a stated reason and no score; a `3/1/1/0` weighting reported back and changing POOR's score to 24.000000 while a negative weight, an unknown metric and an incomplete weight set are refused; and a date window excluding the receipt making GOOD unrated again.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `AP` — Accounts Payable (§2.1)

#### Task ID: T-2.AP.01
- **Title**: Supplier invoice entry and GL posting
- **Description**: Record supplier invoices (with tax and currency) against suppliers and, where applicable, PO/GRN references, and post them to the GL through the shared posting interface.
- **Plan source**: §2.1 Core Components (Accounts Payable (AP) – supplier invoices), §4 Phase 2 bullet 3, §3 row "Double-Entry Integrity"
- **Scope**: Invoice capture and its posting. Matching is T-2.MATCH.01; settlement is T-2.AP.04.
- **Variables / Config**: `transaction_currency`, `tax_pack`, account mapping (payables control, expense).
- **Dependencies**: T-2.PROC.01, T-1.ACCT.03
- **Acceptance Criteria**: An invoice posts a balanced entry to the payables control account on the stated date; a duplicate supplier invoice (same supplier, number, amount) is refused; a foreign-currency invoice records rate, foreign and base amounts; the invoice is attributable to the document that produced it.
- **Evidence**: `ERPbackend/app/ap/invoices.py` (new package `app/ap/`) — `SupplierInvoice`/`SupplierInvoiceLine` with the duplicate rule as both a module check and a database constraint (`uq_supplier_invoice_duplicate` on company + supplier + the supplier's own reference + gross), the due date derived from T-2.PROC.01's payment terms, and links to the order and receipt behind it. `post_invoice` books through **T-1.ACCT.03's mapping only** (`payables`, `input_tax`, and per line `inventory` for an item or `expense` otherwise), so no account code is fixed in the module; **`SupplierInvoiceSettlement`** is append-only and `open_amount`/`settled_amount` derive what is owed from those rows, which is the one figure T-2.AP.02/`.04`/`.05` will read. Check: `ERPbackend/tests/check_supplier_invoice.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_supplier_invoice.py`, exit 0) — green on all eight: AP-6001 posting 1170.000000 to 2000 on 2026-11-05 with 1000.000000 to `inventory` (stock line), 50.000000 to `expense` (service line) and 120.000000 to `input_tax`, balanced; the due date 2026-12-05 = invoice date + 30 days; the re-keyed invoice refused by name **and** by the database; a USD invoice keeping rate 58.5000000000 and base 12402.000000 exactly derivable; 700.00 settled leaving 470.000000 open while an over-settlement, a zero settlement and settling a **draft** are refused; the settlement table refusing an UPDATE (`append-only`); and an empty invoice, a blank supplier reference and a negative line all refused.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.AP.02
- **Title**: AP aging report
- **Description**: Age open supplier balances by due date and bucketed periods, per supplier and per company.
- **Plan source**: §2.1 Core Components (AP – aging)
- **Scope**: Aging report and its export. Dunning for payables is not named in the plan (dunning belongs to AR).
- **Variables / Config**: `company_id`, `report_schedule`, aging buckets (per company — not stated).
- **Dependencies**: T-2.AP.01
- **Acceptance Criteria**: Aging buckets are configurable and stated on the report; each aged amount traces to an open invoice; the report total equals the AP control account balance; partial settlements reduce the correct bucket.
- **Evidence**: `ERPbackend/app/ap/aging.py` — `aging()` walks T-2.AP.01's **open invoices** (so every figure traces to one: number, supplier, due date, days past due, gross, settled and open are all on the report) and buckets them with `checked_buckets`, which **refuses** any bucket set that would lose an invoice (a gap, an overlap, a wrong start, two open ends, a closed last band, an empty set, a reversed span). A partial settlement needs no special case: the bucket holds `open_amount`, so a payment reduces its own band and a fully settled invoice drops out. `by_supplier()` re-cuts the same rows and `aging_csv()` exports them with the buckets named. Check: `ERPbackend/tests/check_ap_aging.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ap_aging.py`, exit 0) — green on all seven: five invoices aged across `current` 600.000000, `1-30` 150.000000, `31-60` 300.000000, `61-90` 0.000000 and `90+` 400.000000 summing to 1450.000000 — exactly what the invoices' open amounts add to; a 50.00 partial settlement leaving 150.000000 open in its own band and a fully settled invoice absent entirely; three custom bands giving 600/150/700 on the same total; all seven losing bucket sets refused with their reasons printed; ACME 750.000000 + BOREAL 700.000000 adding back to the company total; and a 13-line export carrying one row per aged invoice plus the bucket totals. `Report.compare_to_control` (added 2026-09-30) states the payables control account **beside** the subledger total, per currency — read from the ledger through T-2.AP.05's `control_balance`, so the report and the reconciliation cannot disagree about what the control account holds — with the difference and a `balanced` verdict, and `aging_csv` prints both against the bucket totals. Case 8 of the check was added on top of the seven: the settlements now post their counter-entry the way T-2.AP.04's payment run does (a settlement record alone moves the subledger, not the ledger — the control account still showed the whole invoice as owed until it did), so the report's 1450.000000 is asserted equal to the control account's own 1450.000000, and then 300.00 posted straight to the control account is reported by the next run as a difference of `-300.000000` with `balanced: False` while the subledger total stays put. Re-run 2026-09-30 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ap_aging.py`, exit 0) — green on all eight, the export 15 lines.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.AP.03
- **Title**: Debit notes for supplier returns and adjustments
- **Description**: Issue debit notes against suppliers (returns, price adjustments) with their stock and GL effect.
- **Plan source**: §2.1 Core Components (AP – debit notes)
- **Scope**: Debit note document, stock return where applicable, and posting. Credit notes on the sales side are Phase 3 scope.
- **Variables / Config**: `tax_pack`, `transaction_currency`, `warehouse_hierarchy_levels` (return location).
- **Dependencies**: T-2.AP.01, T-1.INV.05
- **Acceptance Criteria**: A debit note posts the reverse of the corresponding invoice lines and balances; a return reduces stock at the correct location and value; a debit note cannot be raised for more than the invoice's remaining open amount without an explicit override that is recorded.
- **Evidence**: `ERPbackend/app/ap/debit_notes.py` — `DebitNote`/`DebitNoteLine` for the two shapes (`return`/`adjustment`, with the table check forcing a location on a return and none on an adjustment). `post_debit_note` posts **one** entry that is T-2.AP.01's posting with the sides swapped (payables debited, the cost accounts and input tax credited), then for a return writes one stock movement per line with `record_movement` **without** its own GL posting — `transactions.issue` would post a second entry crediting inventory for the same goods, so one economic event is booked once — at the value `value_issue` gives, and it settles the invoice through T-2.AP.01's `settle` so aging and the reconciliation see it without knowing debit notes exist. An over-note is refused unless a reason is recorded; a return from a location that does not hold the quantity, or of a line with no stock item, is refused. Check: `ERPbackend/tests/check_debit_note.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_debit_note.py`, exit 0) — green on all eight: DN-7001 (adjustment) reversing the invoice with payables debited 56.000000, inventory credited 50.000000 and input tax 6.000000, balanced, and dropping the invoice from 1120.000000 to 1064.000000 through a settlement row naming it; a beyond-open note refused and then accepted with a recorded reason, settling the invoice fully; DN-7003 taking 4 back out of `MAIN-B1` (10.000000 → 6.000000, value 1000.000000 → 600.000000) with **exactly one** GL entry and one movement at source_type `debit_note`; a return from an empty bin and a return of a service line both refused; a draft invoice, an unknown kind, a return with no location, an adjustment with one and an empty note refused; and a posted note never posted twice with a repeated number refused, the refused return staying a draft that moved nothing.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.AP.04
- **Title**: Payment batches and payment run with settlement posting
- **Description**: Select open invoices into a payment batch, approve it, execute the payment run, and post the settlement — supporting partial and full settlement.
- **Plan source**: §2.1 Core Components (AP – payment batches), §4 Phase 2 bullet 3
- **Scope**: Batch selection, approval, execution and posting. Bank file generation for payables uses `bank_file_format` from the pack; automated bank transmission is not named in the plan.
- **Variables / Config**: `payment_batch_schedule`, `approval_levels`/`approval_thresholds`, `bank_file_format`, `transaction_currency`.
- **Dependencies**: T-2.AP.01, T-2.MATCH.01
- **Acceptance Criteria**: A batch cannot include an invoice that has failed 3-way matching (metric-facing, see T-2.MATCH.02) or is otherwise on hold; execution posts one balanced entry per settled invoice; partial settlement leaves the correct remaining balance; a batch cannot be executed twice; removing an invoice after execution is impossible.
- **Evidence**: `ERPbackend/app/ap/payments.py` — `PaymentBatch`/`PaymentBatchLine` with the approval routed through **T-0.WF.01** (`doc_type = "payment_batch"`), `create_batch` checking every invoice it selects (posted, owing, one currency, and **`require_not_held`** from T-2.MATCH.02), `execute_batch` re-checking the hold **at payment time** as well and posting **one balanced entry per settled invoice** (payables debited, bank credited, through T-1.ACCT.03's mapping) while writing the settlement T-2.AP.01 and T-2.AP.02 read, and `bank_file` writing the pack's own columns (`bank_file_format`) in its order, **refusing** a required column with no value. Check: `ERPbackend/tests/check_payment_batch.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_payment_batch.py`, exit 0) — green on all eight: PB-401 selecting AP-401 + AP-402 for 1456.000000; a draft invoice, a mixed-currency batch, an unknown invoice and a fully settled one refused with their reasons; both invoices settled and two balanced entries posted with the batch's paid total equal to the settlements it wrote; an unapproved batch and a second run refused; a paid line stuck in its batch while a draft releases its invoice; a held invoice refused at build time **and** a hold raised after the build stopping the run; the bank file's own header `payee_name,payee_account,bank_code,amount,reference,purpose` with a supplier holding no bank account stopping it; and a repeated number plus an empty batch refused.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-2.AP.05
- **Title**: AP to GL reconciliation and payables control check
- **Description**: Prove that open supplier balances equal the payables control account, per company and period, and report any difference.
- **Plan source**: §3 row "Double-Entry Integrity", §2.1 Key Capabilities (Automatic double-entry posting from all sub-modules)
- **Scope**: The reconciliation only. No new posting logic.
- **Variables / Config**: `company_id`, `base_currency`.
- **Dependencies**: T-2.AP.01, T-2.AP.04
- **Acceptance Criteria**: Subledger open balance equals the control account to currency precision for any period; an injected mismatch is reported rather than swallowed; the reconciliation runs per company.
- **Evidence**: `ERPbackend/app/ap/reconciliation.py` — `reconcile()` compares the subledger's **derived** open balance (`open_amount`, so a payment or a debit note moves both sides by construction) against the payables control account read from `journal_line` joined to its entry, **per currency**, because that is the unit both sides were posted in and a base-currency comparison would need a rate neither side stored; it returns the difference and a verdict and **corrects nothing**. Check: `ERPbackend/tests/check_ap_reconciliation.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ap_reconciliation.py`, exit 0) — green on all seven: invoice and control account both reading 1120.000000; a 500.00 settlement leaving both at 620.000000 and a 20.00 debit note taking both to 600.000000; PHP 600.000000 and USD 224.000000 compared separately, each against its own entry; an injected 300.00 straight to the control account reported as `a difference of -300.000000` rather than absorbed; a period starting after the last posting reading 0.000000 against an open 600.000000; and an empty currency reading nil on both sides while a company with no invoices states there is nothing to compare. `reconcile` also states a period as a **window on a balance** (2026-09-30): with `start` given, the control side is re-stated as the `opening` balance plus the `movement` over the period — both named in the row — instead of the bare period movement, which is what made a period read as a difference that was not a disagreement. Case 6 now proves it as a balance: opening 620.000000 + movement 280.000000 = 900.000000, the identical control figure and `-300.000000` difference the un-windowed run reports, plus a second company's own period reconciling at 5000.000000 — so the criterion holds **for any period**, not only for one that happens to start at the first posting. Re-run 2026-09-30 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ap_reconciliation.py`, exit 0) — green on all seven.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `MATCH` — 3-way matching (§2.3)

#### Task ID: T-2.MATCH.01
- **Title**: 3-way match engine (PO ↔ GRN ↔ Supplier Invoice)
- **Description**: Compare purchase order, goods receipt and supplier invoice line by line with configurable tolerances, and mark each invoice as matched, partially matched or failed. This owns §6 metric 3 (> 95 % match rate).
- **Plan source**: §2.3 Core Components (3-way matching (PO ↔ GRN ↔ Supplier Invoice)), §2.3 Key Capabilities, §4 Phase 2 bullet 2 and exit criteria, §6 metric 3
- **Scope**: The comparison and its verdicts plus match-rate measurement. Held-invoice handling is T-2.MATCH.02; payment selection is T-2.AP.04.
- **Variables / Config**: `three_way_match_tolerance`, `three_way_match_target` (> 95 %), `tax_pack` (comparison basis), `transaction_currency`.
- **Dependencies**: T-2.PROC.07, T-2.AP.01
- **Acceptance Criteria**: Quantity, price and tax are compared per line within configured tolerances and each is reported separately when exceeded; the verdict is explainable (which line, which value, which tolerance); the match rate is measured over a rolling period and reported against the > 95 % target; a tolerance cannot be set so loose that a wrong-price invoice passes silently.
- **Evidence**: `ERPbackend/app/matching.py` — `match_invoice` compares an invoice **line by line** on three dimensions (quantity invoiced against received, price invoiced against ordered, tax invoiced against what T-2.PROC.08's rule implies for the ordered value) and returns `matched`/`partial`/`failed` with a **findings** payload naming the line, the dimension, the expected value, the stated value and the tolerance exceeded; a line with no ordered/received line behind it is a finding of its own. `MatchTolerance` is a per-company row whose values are **capped at 5 %** (the plan states none, so the ceiling is a platform decision) and `MatchRun` is append-only so "what did the match say then" stays answerable. `match_rate` measures §6 metric 3 over each invoice's **latest** run. Check: `ERPbackend/tests/check_three_way_match.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_three_way_match.py`, exit 0) — green on all eight: a clean AP-801 matched on all three; a 10-short quantity, a 20 %-over price (with the tax outside its own band too) and a 0.5 %-over price inside the 1 % band giving three different verdicts; the explanation reading `line 1 price expected 100.000000 but was 120.000000 (tolerance 1.0000%)`; an unordered line failing with a linkage finding; a 50 % tolerance refused against the ceiling; a rate of 40.0000 % over 5 invoices (2 matched, 2 partial, 1 failed) **unmoved by re-running one invoice**; a run refusing an UPDATE while the first run stayed readable after a second was appended; and an unposted invoice refusing to be matched.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Accounting)
- **Status**: DONE

#### Task ID: T-2.MATCH.02
- **Title**: Match exception and hold/release workflow
- **Description**: Hold invoices that fail matching, route exceptions for resolution, and allow release only through an authorised, audited override.
- **Plan source**: §2.3 Core Components (3-way matching), §3 row "Workflows & Approvals", §1 principle 6
- **Scope**: Hold/release and exception resolution. The tolerance maths is T-2.MATCH.01.
- **Variables / Config**: `approval_levels`, `approval_thresholds`, `rbac_roles` (who may override).
- **Dependencies**: T-2.MATCH.01, T-0.WF.01
- **Acceptance Criteria**: A failed invoice cannot enter a payment batch while held; release requires the configured permission and records actor, reason and the exception it resolves; an override does not alter the underlying match verdict (the exception stays visible for reporting); the match-rate metric counts overrides separately from clean matches.
- **Evidence**: `ERPbackend/app/matching.py` (the hold/release half) — `MatchHold` (one open hold per invoice, `uq_match_hold_open`, carrying the **run it is held against** so the override can be read against the verdict it overrode), `hold_invoice` refusing a cleanly matched invoice or an unmatched one, `hold_failed_matches` for the nightly sweep (skipping an exception already raised **for that verdict**, so a resolved one is not re-opened), `require_not_held` as the gate T-2.AP.04's payment selection calls, and `release`, which authorises through **T-0.SEC.01's `require`** (`match.override`) rather than deciding seniority in code and records actor, date and reason. `override_count` counts releases separately from clean matches. Check: `ERPbackend/tests/check_match_hold.py`, run 2026-09-29 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_match_hold.py`, exit 0) — green on all eight: AP-902 held against its partial run while the matched AP-901 was refused a hold; `require_not_held` refusing the held invoice by name; a clerk refused (`'clerk.jo' may not 'match.override'`) while a controller released it; the release recording fin.ada / 2026-11-30 / its reason, with a reasonless release refused; **no new `MatchRun`** written and the partial verdict byte-identical after the override; the rate staying 50.0000 % with the override counted separately; the sweep holding only the unresolved AP-903 and the invoice leaving the hold on release; and holding twice, holding without a reason, releasing a release and holding an unmatched invoice all refused.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

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
- **Evidence**: `ERPbackend/tests/check_phase2_exit.py`, re-run 2026-09-30 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_phase2_exit.py`, exit 0) — all five criteria substantiated in the tree:
  - **the cycle, with no re-keying** — `REQ-G1` (11000.000000) approved through its own chain, issued as `RFQ-G1` to ACME, BOREAL and CHIRP, answered by ACME (both lines) and BOREAL (both lines, **late**), silent from CHIRP. The **comparative statement** is now built inside the cycle by reading `GET /rfqs/RFQ-G1` — the payload T-2.PROC.04 renders the matrix from in the frontend repository — and it states all three invitees side by side on its labelled basis (`VAT-IN-12`, 12 %): ACME 100.000000/50.000000, BOREAL 48.000000 on line 2, CHIRP blank with `responded: false`. `award` then reads ACME's and BOREAL's own quotes into `PO-G1` (10750.000000) and `PO-G2` (240.000000), and the hand-off is **proved rather than restated**: every `purchase_order_line.unit_price` equals the price the statement carried for the RFQ line behind it (`rfq_line_id`), so the PO's own line numbering cannot hide a re-key. Both orders approved; received as `GRN-G1` (115 accepted, 10750.000000) and `GRN-G2` (5 accepted, 1 **rejected**); invoiced as `AP-G1` and `AP-G2`; matched; paid by `PB-G1` (12362.560000) which settled both.
  - **the match rate, measured** (§6 metric 3) — 100.0000 % over the clean month (`met: True`) and 50.0000 % over the whole dataset (`met: False`) reported against the 95 % target, so the metric is shown to be mettable and not merely computed
  - **the postings and the control account** (§6 metrics 1–2) — the T-0.CORE.02 ledger gate returns 0 over the **stored** ledger for the whole cycle, and `reconcile` reports the AP subledger equal to the payables control account (both 0.000000 after settlement). The gate now also reads T-2.AP.02's `aging()` over the cycle **before** the batch settles it: 12362.560000, stated against the control account's own 12362.560000 (difference 0.000000) and equal to the batch's total — so the aging report and the control account agree on this data too. T-2.AP.05's as-of/opening-balance cases are proved in its own check (case 6), added with it.
  - **a failed match blocks payment** — `AP-G2` was held, refused by `require_not_held` and refused entry into a payment batch twice, then released by an authorised override recorded against fin.ada / 2026-12-18 / its reason, and counted as an override rather than a clean match
  - **supplier tax per the pack** — `VAT-IN-12` at 12 % resolved from the installed pack for `purchase_order`, `goods_receipt` and `supplier_invoice` alike, `tax_on(10750)` = 1290.000000, the figure AP-G1 carries, and the T-2.PROC.08 check scans that module for a hard-coded market
  **Finding closed at the source, not worked around**: the Phase 1 seed (`tests/seed.py`) mapped a goods receipt's counterpart (`stock_receipt`) to the payables control account (`2000`), and the gate had installed a substitute account of its own (`2050`, invented — it is not in the pack) so the control account would reconcile. The seed now maps `stock_receipt` to the pack's own **`2010` Goods Received Not Invoiced**, which the pack nests under `2000`, so a receipt credits 2010 and a supplier invoice's received line debits it back through the same key (T-2.AP.01 books that line to `GRNI_KEY`) — one liability, booked once. The gate installs **no** mapping of its own and asserts the 48.000000 overbilling debit sits in `2010`; `tests/check_supplier_invoice.py` uses the seed's mapping likewise, its own invented `2050` deleted. The canonical default is therefore what the checks exercise, and the whole backend sweep — 44 checks plus the ledger-integrity gate — is green on it.
  The **cross-repository half** of criterion 1 is the frontend's own `tests/comparison.test.ts` over this same payload (T-2.PROC.04) plus the byte-identical vendored contract; the two repositories are separate and neither pipeline needs its sibling, so the gate itself stays in the backend repository. Verified 2026-09-30: `ERPfrontend npm run check` green on `main` at `1994fe4` ("the vendored contract is the backend's published artifact, byte for byte"), and this batch changed no part of the published contract.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: DONE

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
- **Evidence**: **DONE 2026-09-30** — `ERPbackend/app/sales/customers.py` (new package `app/sales/`) + `ERPbackend/tests/check_customer_master.py`, run green against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_customer_master.py`, exit 0), then the whole backend sweep re-run green on the same tree (47/47 — 46 checks incl. this one, plus the ledger-integrity gate). The check substantiates every criterion in the tree:
  - **a dual-role party** — `create_customer` on a party that is already a supplier (created through T-2.PROC.01's own `create_supplier`) adds the **customer role to the same party**: one `party` row, two `party_role` rows, one tax number, and both profiles share one `party_id`. A second `create_customer` for that party is refused by name (`DuplicateCustomerError`).
  - **the two credit-limit states distinguished** — `credit_limit` is a nullable `Numeric(20,6)`: `NULL` is *no limit agreed*, `0` is *no credit at all*, and `25000.50` is a real ceiling. All three are stored and read back through `credit_limit_of()` differently (the check asserts `None` · `Decimal("0")` · `Decimal("25000.50")`), withdrawing with `None` returns to *unset* rather than zero, a negative and a **float** limit are refused at entry (`InvalidCustomerDataError`), and the database refuses a negative one written by hand past the function (`ck_customer_credit_limit`).
  - **addresses reusable across documents** — `customer_address` rows are referenced **by id**: the check's two probe documents point at the *same* address row and both resolve it. At most one primary stands **per kind** (a partial unique index on `(customer_id, kind) WHERE is_primary`), so a new primary billing address demotes the previous one while the shipping address keeps its own flag, and the database refuses a hand-written second primary of a kind. `add_address` validates at entry (blank line, unknown kind, a country that is not ISO 3166-1 alpha-2 refused); `add_contact` refuses a malformed e-mail; an unregistered currency is refused through T-1.ACCT.05's own master.
  - **referenced customers cannot be deleted** — `deny_hard_delete` refuses `DELETE` ("… is a master"); `retire_customer` refuses while a document names the customer, scanning the schema for foreign keys into `customer` (the check's `probe_sales_document` is what fires, and `documents_naming` picks up a real document table the day it appears) and marks the row otherwise; a party without the customer role is not found by `customer_by_code`.
  **Not in scope, deliberately**: enforcement of the limit (T-3.AR.06 computes exposure, T-3.SALES.04 makes the order-time decision) and the sale-side documents themselves. `app/api.py` gained no endpoint and the published contract is unchanged, so the frontend repository needed no commit for this id.
  **Review (2026-09-30), PR paws1234/ERPbackend#10 — 2 findings, both fixed and re-verified**: a **non-finite** credit limit was accepted or blew up — `Decimal("Infinity")` passed the sign test outright and `Decimal("NaN")` raised an uncaught `InvalidOperation` *out of the comparison itself* rather than the module's named refusal — so an `is_finite()` guard now runs before the sign test; and the country check, which accepted any two characters including `12`, is now a two-letter format guard. Both cases were added to `tests/check_customer_master.py` (§4 and §5) and the check is green. The country guard's ceiling is stated in the module rather than hidden: a well-formed but *unassigned* code (`ZZ`) still passes, and the upgrade path is the localization pack's own country list.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.SALES.02
- **Title**: Lead and opportunity pipeline (Kanban)
- **Description**: Track leads and opportunities, move them through configurable pipeline stages on a Kanban board, and record value, owner and expected close date.
- **Plan source**: §2.5 Core Components (Lead & Opportunity pipeline (Kanban))
- **Scope**: Pipeline stages, movement and board. No marketing automation or campaign attribution (not named in the plan).
- **Variables / Config**: pipeline stages (per company — plan names no stages), `rbac_roles`, `company_id`.
- **Dependencies**: T-3.SALES.01, T-0.SEC.01
- **Acceptance Criteria**: Stages are configurable without code change; moving a card records who moved it and when; a won opportunity can be converted to a quotation with the customer details carried over (note: the pipeline slot in §4 Phase 3 is inferred — see Open Questions); a lost opportunity records a reason; the board respects field-level permissions so values can be hidden from some roles.
- **Evidence**: **DONE 2026-09-30 (second pass)** — the domain layer landed under PR paws1234/ERPbackend#10; this pass closed the gap the id was reopened for, so the board is **drivable** through the published API instead of only readable. `ERPbackend/app/sales/pipeline.py` (the domain, plus `card_payload`, `opportunity_by_id` and `latest_move`), `app/sales/quotations.py` (the header), `tests/check_pipeline.py` (§1–§9) and the frontend read path in `ERPfrontend/lib/pipeline.ts` + `app/pipeline/page.tsx` + `tests/pipeline.test.ts`. What this pass added:
  - **the five missing API paths, in `app/api.py`**: `POST {BASE}/pipeline/stages` (`pipeline.configure`), `POST {BASE}/opportunities`, `.../{id}/moves`, `.../{id}/loss` (`opportunity.write`) and `.../{id}/quotation` (`opportunity.write` **and** `quotation.write`, because that call acts on the opportunity and creates a quotation). A board card now carries its **`id`** — without it the board could not be acted on at all, and the payload filters it like every other field, so a role that may not read it gets a board it can look at but not drive rather than a button that fails.
  - **the actor and the instant come from the request, not the body**: `X-Actor` and the server's clock. `PipelineMoveIn` has no `at` field, and §9c asserts the recorded `moved_at` is not older than the moment the test made the call.
  - **a mutation's answer is filtered like the board** — `_opportunity_out` reads a card through the same `card_payload` and the same `hidden_fields`, so a mutation can never become a way of reading a value the board withholds (§9g creates a card as the restricted viewer and asserts `value` is absent from the answer).
  - **the Kanban interactions, in the frontend**: `app/pipeline/actions.ts` (four **server actions**, so the browser never holds the identity the backend reads and never calls the API itself) + `app/pipeline/board-actions.tsx` (the new-deal form and, per card, move / mark-lost / convert, each showing the refusal the backend gave) + `lib/api.ts`'s four mutations + `movableStages()`/`wonStages()`. The loss column is deliberately **not** a move target — ending a deal is the action that requires a reason — and a card whose `id` the role may not read gets no controls.
  - **one new domain guard (§4c)**: `create_opportunity` now refuses to open a card **in a loss column**. Losing is a move with a reason, so a card created there would end before it began and say nothing about why; the API's `stage` field made that reachable, and it is closed in the shared function rather than in the endpoint.
  - **one inconsistency the cold read caught, fixed in `card_payload`**: a card was rendered from the pending object's own `Decimal("5000.00")` while the same card read back off the board said `5000.000000`, so a mutation's answer drifted from the board's. Both render at `MONEY_SCALE` now.
  The evidence already in place substantiates:
  - **stages configurable without code change** — `pipeline_stage` rows, with `is_won`/`is_lost` carrying the *meaning* while the names stay the company's; a company defines its own columns, inserts one between two others, and a duplicate name or position is refused by name (`DuplicateStageError`) while the database's own constraints hold too (`ck_pipeline_stage_won_or_lost` evidence a both-won-and-lost stage written by hand). A company with no board is refused an opportunity rather than given a default column, and a stage anything still references — a card standing in it **or** a recorded move naming it — cannot be removed (`StageInUseError`).
  - **moving a card records who and when** — every movement writes an `opportunity_move` row: the from-stage, the to-stage, the actor and the instant, and `create_opportunity` records the opening move so a deal's history starts where it was created. The check reads the trail back in order and asserts the actor and timestamp.
  - **a won opportunity converts once, with the customer carried over** — `convert_to_quotation` refuses a card in a non-won stage (`NotWonError`), and the quotation it produces links to the customer **row** (`customer_id`) and back to the opportunity; a second conversion is refused (`AlreadyConvertedError`) and the schema's own partial unique index (`uq_quotation_opportunity`) refuses a hand-written second quotation for one win. `app/sales/quotations.py` holds the header only — T-3.SALES.03, which depends on this task, adds the priced lines, validity and the order conversion.
  - **a lost opportunity records a reason** — moving into a loss column without one is refused (`LostReasonRequiredError`), with one it is recorded on both the move and the card, and the card's `closed_at` is stamped.
  - **the board respects field-level permissions** — `board()` filters each card through T-0.SEC.01's `readable_fields`, so a restricted field is **absent** rather than nulled. Proved twice: backend (`tests/check_pipeline.py` gives a junior role `can_read=False` on `opportunity.value` and asserts the key is missing from every card while an unrestricted subject still sees it) and frontend (`lib/pipeline.ts` reports such cards as `withheld` and never folds them into the total — the test asserts a hidden card is not counted as zero while an explicit `"0"` is).
  **Verified this pass**: the backend pipeline's own checks, run locally exactly as `.github/workflows/backend.yml` runs them (the loop over `tests/check_*.py`, excluding `check_backend_image.py`/`check_compose_stack.py` for the same reasons the pipeline excludes them, plus the ledger-integrity gate last) — **46 checks + the gate, all green.** `check_api_conventions.py` re-proves the served contract equals the published artifact and that all **178** documented fields are typed with optional ⇒ nullable; it is what caught `PipelineStageIn.is_won`/`is_lost` as optional-but-not-nullable during the cold read (now `bool | None`). Frontend: `npm run check` green (no database driver, the vendored contract byte-identical, the generated types current, `tsc` clean, the removed-field build failure, `next build`) and `tests/pipeline.test.ts` **9/9** — up from 8, the new case covering the two column helpers. Contract republished and re-vendored byte for byte.
  **Widened by a later id in the same batch**: T-3.SALES.03 extended `QuotationOut` — this task's conversion endpoint (`POST …/{id}/quotation`) answers with the whole document now (its lines, window, expiry and total) rather than the header alone, and both endpoints build it through one `_quotation_out`. The §9e assertions still hold unchanged; the artifact was republished again by that task, so the diff for this id's contract hunk no longer shows the header-only shape on its own.
  **Routing finding**: `T-3.SALES.02` belongs in the map's "Both — client in the frontend repo, API and postings in the backend repo" category because its acceptance criteria span backend domain rules/API and frontend interactions.
  **Review (2026-09-30), PR paws1234/ERPbackend#10 — 7 findings, all fixed and re-verified.** Two were **high**, both tenant holes: `create_opportunity` never checked `customer.company_id` and accepted an explicit `stage` from another company's board, and `create_quotation` took `company_id`, `customer_id` and `opportunity_id` independently — so a company-A row could name company-B data and disappear from both boards. Both are refused by name now, proved in §8a and §8b of the check. Five were medium: `opportunity_move` is now **`append_only`** (§8c) — it is the trail auditors read, and `approval_decision` and `match_run` were already registered, so leaving it out was an inconsistency with the house convention rather than a missing idea; moving a won or lost card back to a normal stage now **clears** `closed_at` and the stale `lost_reason` instead of leaving an open deal reading as finished (§8d); `convert_to_quotation` now passes the **customer's own currency**, so a USD customer no longer gets a base-currency quotation while the docstring claimed the details were carried across (§8e); the board reads the subject's field restrictions **once per request** instead of once per card, through a new `hidden_fields()` in `app/security.py` that `readable_fields()` now also uses (a pure extraction, flagged because that module is T-0.SEC.01's); and `PipelineCardOut.name` is now **optional** in the contract, because a restriction can be stated against any field and a 200 that violates its own schema is worse than an honest optional. Contract republished and re-vendored byte for byte, types regenerated, and `tests/pipeline.test.ts` gained the withheld-name case.
  **Review after the push (2026-09-30)**: the batch's own review pass found **nothing in this id's own code**. Its one finding that touches a function this id wrote — `create_quotation` accepting a currency the company has not registered — is recorded and fixed under `T-3.SALES.03`, whose `POST /quotations` is the only caller able to pass an arbitrary code. The automated review was asked for and refused by the remote (`422`, not a collaborator); see that id's record.
- **Estimated Effort**: M
- **Owner Role**: Frontend Engineer
- **Status**: DONE

#### Task ID: T-3.SALES.03
- **Title**: Quotations and quotation-to-order conversion
- **Description**: Create quotations with priced lines and validity, and convert an accepted quotation into a sales order without re-keying.
- **Plan source**: §2.5 Key Capabilities (Quotation → Order conversion), §4 Phase 3 bullet 2
- **Scope**: Quotation document and the conversion path. Pricing comes from T-3.SALES.06/07.
- **Variables / Config**: `transaction_currency`, `tax_pack`, quotation validity (plan states none), `pricing_rule_dimensions`.
- **Dependencies**: T-3.SALES.02
- **Acceptance Criteria**: A quotation prices its lines from the pricing engine at the time of quote and records which rule applied; an expired quotation cannot be converted without re-pricing; conversion produces an order with identical lines and links back to the quotation; a quotation cannot be converted twice.
- **Evidence**: **DONE 2026-09-30** — the quotation is a document now, not only a header. **Backend only** (the ledger routes `T-3.SALES.03` to the backend repository). New `ERPbackend/app/sales/orders.py` (the order, its lines and the conversion); `app/sales/quotations.py` gained `QuotationLine`, `Quotation.valid_until`, `add_line`, `reprice_quotation`, `expired`/`refuse_if_expired`, `line_amount` and `MONEY_SCALE`; `app/api.py` gained `POST {BASE}/quotations`, `GET {BASE}/quotations/{number}`, `POST …/reprice` and `POST …/order` (`quotation.read`/`quotation.write`, plus `order.write` for the conversion, since it creates T-3.SALES.04's document); `ERPbackend/tests/check_quotations.py` is the check, green on all six sections.
  - **a quotation prices its lines when it is written, recording the rule that applied** — `add_line` stores the price, the **`rule_code`** that produced it and the day it was fixed (**`priced_on`**, defaulting to the day the line is written, which the API check asserts against the clock rather than a hard-coded day). A price a person stated records `rule_code = NULL` — an honest null, not an invented code — and §1 proves both cases side by side.
  - **an expired quotation cannot be converted without re-pricing** — `valid_until` is the window (null means no window, so a quotation with none never expires — the plan names no default and none is invented); `convert_quotation_to_order` refuses a closed window through `refuse_if_expired` and the refusal names re-pricing as the fix; §3 proves the card is *not* ordered by the refused call, and that the same conversion succeeds after `reprice_quotation`. §6 drives the same refusal and recovery over the API.
  - **conversion produces an order with identical lines, linked back** — `convert_quotation_to_order` copies line_no, description, item, quantity, uom, unit price, rule and `priced_on` verbatim; §4 compares the two line sets field for field and asserts the totals agree and that `order.quotation_id` names the quotation.
  - **a quotation cannot be converted twice** — refused by the service (`DuplicateOrderError`) and by the schema's own partial unique index `uq_sales_order_quotation` for a hand-written second order (§5), the same shape T-3.SALES.02 used for `uq_quotation_opportunity`. A converted quotation is no longer an offer: it can neither be re-priced nor have a line added (`ConvertedQuotationError`, also §5).
  - **re-pricing restates every line, with the window the new prices hold for** — a partial re-price, a re-price naming a line that is not there, and a window that has already closed are each refused by name (§2a); the successful re-price moves every `priced_on`, restates the rule, and sets the new window (§2b, including the same-day edge).
  - **the whole path is drivable through the published API**, refusals included (§6): create, read, re-price, order, the second conversion, the expiry refusal and the recovery, and a stranger's 403.
  **Finding — the pricing-engine criterion is partly downstream of its own task.** This task's first criterion names *the pricing engine* (T-3.SALES.06/07), and T-3.SALES.06's own criteria also claim "a quote records the rule that applied so it can be reproduced later" — but SALES.06 depends on SALES.04 which depends on this task, so the engine cannot exist yet. Read as: this task owns the **place** (a priced line carrying `rule_code` and `priced_on`) and the engine fills it. Prices are therefore **stated, not resolved**, and the ceiling is marked with a `ponytail` in `app/sales/quotations.py` naming the upgrade path (the engine calls the same two functions with the price and rule it resolved, so neither changes when it lands).
  **Two defects found while writing the check, both fixed in this task's diff**: the `timezone` import was missing from `app/api.py` (a 500 on the first request — caught by the check, not by review); and `line_amount`/`order_total` multiplied two six-decimal operands and returned **twelve** decimals (`Decimal` scales add), so a quotation total read `619.970000000000` against the platform's money scale — both now quantize at `MONEY_SCALE`, and the API layer stopped carrying its own duplicate `_exact` helper because the domain does it once.
  **Review after the push (2026-09-30), on the open PR — 5 findings, all real, all fixed under this id; commits `ad294fc` and `adff0fc`.** The request's own review was **asked for and refused by the remote**: `POST …/requested_reviewers` with the Copilot reviewer answers `422 Reviews may only be requested from collaborators` with the token these runs hold, and the repository carries no automatic-review rule, so the automated review has to be asked for from the PR's *Reviewers* menu. The read was therefore done by hand, on `git diff main...HEAD` as the reviewer meets it, and it found:
  - **an unregistered currency was accepted.** `create_quotation` stored the code as given while the customer master validates against T-1.ACCT.05's master — and this id's `POST /quotations` is the only caller able to pass an arbitrary code, so a quotation could name a currency nobody could convert, price or pay. Now resolved through `currency_by_code` and refused by name (`unknown_currency`); §1 proves it.
  - **a re-price could leave a stale rule behind.** The rule was restated only when the caller named one, so re-pricing a line with no `rule_code` kept the rule that produced its *old* price — precisely what the docstring claimed not to allow. A re-price now always restates the rule (§2b).
  - **a quotation with no lines could be created.** The boundary took `lines: []`, and because no endpoint adds a line to an existing quotation, the store held a document that could never become an order. `Field(min_length=1)` refuses it (§6) — which is what the docstring already claimed.
  - **two refusals existed with nothing asserting them** — `IncompleteOrderError` (the service is a caller too, even though the boundary refuses an empty list) and `DuplicateQuotationError` (a number already used). Both are exercised now (§4b, §6), because an unexercised refusal is a claim rather than evidence.
  The three code findings landed in `ad294fc` (this id's diff) and the two assertion findings in `adff0fc`; the artifact changed, so the frontend re-vendored it in `20214b9`. Pushed history was never rewritten, each follow-up commit carries the id that owns its finding, and the whole sweep was re-run after each — **47 checks + the gate, green** — with the remote pipelines green on every pushed head.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.SALES.04
- **Title**: Sales orders with credit checking
- **Description**: Manage sales orders through confirmation, and apply the customer's credit check at order time according to the configured mode.
- **Plan source**: §2.5 Core Components (Sales Orders & Fulfillment (credit checks, pick lists, shipping)), §4 Phase 3 bullet 1/2
- **Scope**: Order document and the order-time credit decision. Exposure calculation across open AR is T-3.AR.06; fulfilment is T-3.SALES.05.
- **Variables / Config**: `credit_limit`, `credit_check_mode` (`off`/`warn`/`block`), `transaction_currency`.
- **Dependencies**: T-3.SALES.01, T-3.SALES.03
- **Acceptance Criteria**: In `block` mode an order that breaches the limit cannot be confirmed; in `warn` mode it can be confirmed only with a recorded acknowledgement; the decision shows the limit, the current exposure and the order value that produced it; changing `credit_check_mode` does not alter historical decisions.
- **Evidence**: **DONE 2026-10-07** — the order's lifecycle and the order-time credit decision, in the **backend repository**. `app/sales/orders.py` (the order gains `status`/`confirmed_at`/`confirmed_by`, plus the append-only `credit_decision` table and `confirm_order`), `app/company.py` (the per-company `credit_check_mode` column, `set_credit_check_mode`, `credit_check_mode_of`), `app/api.py` (`POST {BASE}/companies/current/credit-check-mode`, `POST {BASE}/sales-orders/{number}/confirm`, `GET {BASE}/sales-orders/{number}`, `ConfirmOrderIn`/`CreditDecisionOut`, and `OrderOut` carrying the lifecycle and the decision), the republished `contract/v1/openapi.json`, and the one check `tests/check_credit_check.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_credit_check.py`, exit 0), green on all seven sections, which substantiate every criterion in the tree:
  - **`block` refuses the breach and only the breach** — the order that would leave the customer owing 1103.00 against an agreed limit of 1000.00 is refused by the primitive (`confirming 'SO-2' would take the customer to 1103.000000, over the agreed limit 1000.000000: refused while the mode is 'block'`), the order is still `draft` and **no decision was recorded**; the same mode confirms the order at exposure 500.00 (after 803.00), so the refusal is the breach and not the mode.
  - **`warn` confirms only with a recorded acknowledgement** — refused without one (`CreditAcknowledgementRequired`), and with it the decision is written with `acknowledged_by='maria'`, so a breach somebody accepted is attributable. The acknowledgement is an explicit act (`acknowledge_breach`), never inferred from the request merely arriving.
  - **the decision shows the limit, the exposure and the order value that produced it** — the recorded row reads back `mode='warn'`, `limit_amount=1000.000000`, `exposure=800.000000`, `order_value=303.000000`, and the API states the same numbers plus the `exposure_after` it computed (1103.000000). A customer with **no limit agreed** records the honest `null` (not zero) and is not judged against one.
  - **changing the mode does not alter historical decisions** — after `set_credit_check_mode(…, mode='off')` **and** the limit cut to 10.00, the decision still reads `warn`/1000.000000/800.000000/`breached`/`maria`: the mode and the limit are recorded values, not pointers. It is append-only too — a hand-written `UPDATE` and `DELETE` are both refused at COMMIT by T-0.AUDIT.01's trigger (`append-only`).
  - **an unstated mode is a refusal, not a silent policy** — this is the decision recorded in the Variables table and plan §8, and it is why the column is nullable with **no default**: `off` is the policy "check nothing", unstated is no policy at all, and the two are asserted apart. The company master already carried `costing_method` for exactly this shape of per-company policy, and keeping the column nullable meant **none** of the 50 `Company(...)` constructions in the existing checks had to change.
  - **the exposure is validated, not trusted** — a float, an infinity and a negative are each refused rather than compared (`CreditError`), and the order is not confirmed by the attempt. `credit_check_mode` is one of the three modes or a refusal naming them.
  - **the whole path is drivable through the published API** — the mode is stated with `POST …/credit-check-mode` (204/200, and a bad mode is `422 unknown_credit_check_mode`), a breach is refused in the platform's one error shape (`422 order_error`), the confirmation answers with the decision, and `GET …/sales-orders/SO-5` serves that same decision back unchanged.
  **Path taken (user decision, 2026-10-07)**: option (a) from this task's own recorded stop — the credit decision is landed against an exposure the **caller states**, until T-3.AR.06 computes it across open AR. A `warn`/`block` outcome computed from an exposure that understated what a customer already owes would look exactly like a pass, which is why the number is stated by the caller and recorded verbatim rather than guessed here. **The second open item in that record is answered by the code rather than invented**: `credit_check_mode` lives on the `Company` master, following `costing_method`'s precedent for a per-company policy.
  **Not in scope, deliberately**: the live exposure itself (T-3.AR.06 — this task's `confirm_order` takes the number and records it), fulfilment (T-3.SALES.05), and creating a **standalone** order for a customer with no quotation (the plan names no such path in Phase 3; the partial unique index on `quotation_id` already leaves room for it). **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**48/48**, the two excluded for the reasons the workflow states) and `tests/check_ledger_integrity.py` green; the frontend's `npm run check` green on all seven assertions after the contract was re-published and re-vendored (`ERPfrontend/contract/v1/openapi.json` byte-identical, `lib/contract.d.ts` regenerated).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.SALES.05
- **Title**: Fulfilment — pick lists, shipping and stock issue
- **Description**: Generate pick lists from confirmed orders, record picked quantities and ship, issuing stock and posting the cost side of the sale through the shared posting interface.
- **Plan source**: §2.5 Core Components (Sales Orders & Fulfillment … pick lists, shipping), §4 Phase 3 bullet 2
- **Scope**: Fulfilment execution and its stock/GL effect. Invoicing is T-3.AR.01.
- **Variables / Config**: `warehouse_hierarchy_levels`, `uom_conversion_factor`, `costing_method` (cost of goods issued).
- **Dependencies**: T-3.SALES.04, T-1.INV.05, T-1.ACCT.03
- **Acceptance Criteria**: A pick list covers exactly the confirmed order lines; partial picking and partial shipping leave correct remaining quantities on the order; shipping issues stock at the correct warehouse with the correct valuation; a shipped order cannot be shipped again for the same quantity; every posting balances.
- **Evidence**: **DONE 2026-10-07** — fulfilment: pick lists, shipping and the stock issue behind them, in the **backend repository**. New module `app/sales/fulfilment.py` (`PickList`/`PickListLine`, `Shipment`/`ShipmentLine`, `generate_pick_list`, `record_picked`, `ship_order`, `remaining_quantity`, `pick_list_for`, `pick_lines`, `shipments_for`), `SalesOrderLine.shipped_quantity` in `app/sales/orders.py`, the endpoints `POST {BASE}/sales-orders/{number}/pick-list`, `POST …/pick-list/lines/{line_no}` and `POST …/shipments` with `OrderOut` carrying `pick_list`, `shipments` and each line's `shipped`/`remaining` in `app/api.py`, the republished `contract/v1/openapi.json`, and the one check `tests/check_fulfilment.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_fulfilment.py`, exit 0), **one order fulfilled in two shipments** at the bin's own moving average, which substantiates every criterion in the tree:
  - **a pick list covers exactly the confirmed order lines** — the order of 10 widgets plus a freight line yields a pick list of exactly two lines, each carrying its `item_id` and `uom` from the order line (the freight line's item is honestly `null`); a **draft** order is refused (`sales order 'SO-1' is 'draft'; only a confirmed order is picked`), a second pick list for the same order is refused, and so is a number the company already uses.
  - **partial picking and partial shipping leave correct queryable remainders** — picking 10 of 10 is recorded (and 11 is refused: `line 1 asks for 10.000000; 11 was picked`), then shipping 4 of 10 leaves the line reporting **4 shipped / 6 remaining** and shipping the other 6 leaves it at **10 shipped / 0 remaining**, read from `SalesOrderLine.shipped_quantity` the same way a purchase order line reports `received_quantity`.
  - **shipping issues stock at the correct warehouse with the correct valuation** — the issue is T-1.INV.05's own `issue` against the named leaf bin, so the bin's on-hand falls from 100 to 90 across the two shipments **at the company's moving average** (4 units → 16.00, then 6 units → 24.00, read back through `on_hand`), each `ShipmentLine` points at its own stock ledger entry, and the stock ledger is filed under the shipment (`movements_for_source`). Nothing here restates valuation, UOM conversion or the no-negative-stock rule.
  - **a shipped order cannot be shipped again for the same quantity** — the third shipment is refused (`line 1 still owes 0.000000 of 10.000000`), and **shipping more than the bin holds** is refused by the no-negative-stock rule rather than clamped (`MAIN-B1 holds 90.000000 of 'WIDGET'`), so neither failure is a silent adjustment.
  - **every posting balances** — each shipment's journal entry is read back and its debits asserted equal to its credits (16.00 and 24.00), the cost side landing on the account T-1.INV.05 maps `stock_issue` to; the two-line minimum is asserted with it.
  - **a line that names no stock item is not shipped** — the freight line is refused (`line 2 of order 'SO-2' names no stock item — a service is not issued out of a location`), matching T-2.PROC.09's receiving wording, and a shipment naming no lines is refused rather than stored empty.
  - **the whole path is drivable through the published API** — `POST …/pick-list` (201, exactly the two lines with the item's SKU), `POST …/pick-list/lines/1` (200, `picked_quantity` 10), `POST …/shipments` (201, warehouse `MAIN-B1`, line reporting **4 shipped / 6 remaining**), a pick list for an unconfirmed order refused as `422 fulfilment_error`, and an unknown warehouse refused as `422 location_error` by name. `GET …/sales-orders/SO-3` serves the same remainders back.
  **Not in scope, deliberately**: the invoice (T-3.AR.01 — it references the shipment rather than issuing the stock again), and requiring a pick list before shipping, which neither the plan nor this task's criteria ask for. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**49/49**) and `tests/check_ledger_integrity.py` green; the frontend's `npm run check` green on all seven after the contract was re-published and re-vendored (`ERPfrontend/contract/v1/openapi.json` byte-identical, `lib/contract.d.ts` regenerated).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.SALES.06
- **Title**: Pricing engine — customer tiers and volume rules
- **Description**: Resolve the applicable price for an item/customer/quantity using ordered rules over customer tier and volume, with an explainable outcome.
- **Plan source**: §2.5 Core Components (Pricing Rules & Discounts (customer tiers, volume…)), §4 Phase 3 bullet 4 (Pricing engine)
- **Scope**: The engine and tier/volume dimensions. Campaigns and coupons are T-3.SALES.07.
- **Variables / Config**: `pricing_rule_dimensions` (customer tier, volume), `discount_type`, `transaction_currency`.
- **Dependencies**: T-3.SALES.04
- **Acceptance Criteria**: Overlapping rules resolve deterministically, and the resolution order is stated on the priced line; a quote records the rule that applied so it can be reproduced later; a tier change does not retroactively change already-priced documents; the engine is used by both quotations and orders (one implementation, not two).
- **Evidence**: **DONE 2026-10-07** — the pricing engine: ordered rules over customer tier and volume, in the **backend repository**. New module `app/sales/pricing.py` (`PriceRule`, `define_rule`, `rule_by_code`, `resolve_price`, `PriceDecision`, `price_for`, `_order_key`, `tier_of`, `price_quote_line`), `Customer.tier` + `set_customer_tier` in `app/sales/customers.py`, `rule_priority` on `QuotationLine` and `SalesOrderLine` (copied by the conversion in `app/sales/orders.py`) and in `add_line`, the endpoints `POST {BASE}/price-rules` and `POST {BASE}/price-rules/resolve` with `rule_priority` exposed on both line outputs, the republished `contract/v1/openapi.json`, and the one check `tests/check_pricing.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_pricing.py`, exit 0), **five overlapping rules on one item resolved deterministically**, which substantiates every criterion in the tree:
  - **overlapping rules resolve deterministically, with the winning rule shown** — five rules all match the same widget/gold-tier/60 line, and the winner is `OVERRIDE` at 70.000000 off a 100.00 base; re-running the same query returns the same winner, the same price and the same ordering, so the answer does not depend on the order rows come back in.
  - **the resolution order is stated on the priced line** — the priced line carries the winning rule's `code` **and** the `priority` that decided it (`OVERRIDE`, priority 1), so the line explains itself after the rules behind it have changed. The order itself is fixed and documented in the module: `priority` ascending, then the more specific scope (item before any-item, tier before any-tier), then the tighter volume band, then the code — and `PriceDecision.considered` returns all of them, winner first, so a caller can say which rules lost and why.
  - **a quote records the rule that applied, so it can be reproduced, and a tier change does not restate it** — after the winning rule was edited to 99 % and priority 900, the same query answers `VOL-20` at 80.000000 while the already-priced quotation line still reports `OVERRIDE`, priority 1 and 70.000000. The edit moved the *engine's* answer forward without rewriting a document.
  - **tier and volume are the dimensions, and null is *no constraint*** — a customer sitting in **no** tier is not matched by a tier-scoped rule (it falls to `ITEM-15`, not `GOLD-TIER-10`), a quantity below a band's floor does not match, a **closed** band applies at its ceiling (20 matches `BAND-25`) and not past it (21 does not), and an open band (null ceiling) keeps applying at 50. An item no rule names still gets the any-item rule, and a line with no item at all too.
  - **the engine is one implementation, used by both documents** — the order converted from the priced quotation carries `rule_code`, `rule_priority` and the price unchanged, so the two documents cannot disagree about why the price is what it is. This is the honest reading of the criterion given T-3.SALES.03's decision that an order is the quotation's lines carried across without re-keying.
  - **the whole path is drivable through the published API** — `POST {BASE}/price-rules` files a rule (201, item resolved from its SKU), a wrong `discount_type` is refused as `422 pricing_error` naming the two allowed values, an unknown SKU as `422 item_error`, and `POST {BASE}/price-rules/resolve` answers 88.000000 for a 12 % rule against 100.00 with the full ordering (4 considered).
  **No invented default**: the ledger names `discount_type` as `percent | amount` with its default **Not stated**, so each rule states its own and there is none to inherit. **No invented price list**: the plan names no price-list master and the item master holds no list price, so the engine takes the **base price from the caller** and decides only which rule applies and what it does. The engine and `PriceRule` live in the sales stream because §2.5 owns pricing; POS reads the same rules through `pricing_rule_dimensions`. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**50/50**) and `tests/check_ledger_integrity.py` green; the frontend's `npm run check` green on all seven after the contract was re-published and re-vendored (`ERPfrontend/contract/v1/openapi.json` byte-identical, `lib/contract.d.ts` regenerated).
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.SALES.07
- **Title**: Campaigns and coupons
- **Description**: Add campaign-scoped and coupon-code discounts on top of the pricing engine, with validity windows and usage limits.
- **Plan source**: §2.5 Core Components (Pricing Rules & Discounts (… campaigns, coupons))
- **Scope**: Campaign and coupon dimensions and redemption. No marketing automation, no customer segmentation beyond tiers (not named).
- **Variables / Config**: `pricing_rule_dimensions` (campaign, coupon), `discount_type`, usage limits and validity (not stated in the plan).
- **Dependencies**: T-3.SALES.06
- **Acceptance Criteria**: A coupon applies only within its validity window and only up to its usage limit; a coupon cannot be stacked beyond the configured allowance; an expired or unknown coupon is rejected with a clear reason; redemption is recorded against the document that used it.
- **Evidence**: **DONE 2026-10-07** — campaigns and coupons, in the **backend repository**. New module `app/sales/campaigns.py` (`Coupon`, `CouponRedemption` — append-only under T-0.AUDIT.01, `define_coupon`, `coupon_by_code`, `coupon_discount`, `redeem_coupon`, `redemptions_for`, `redemption_count`), the **campaign dimension added to T-3.SALES.06's one engine** (`PriceRule.campaign`, `_matches`, `_order_key`, `resolve_price(campaign=…)`, `price_quote_line(campaign=…)`), the endpoints `POST {BASE}/coupons` and `POST {BASE}/coupons/{code}/redeem`, the republished `contract/v1/openapi.json`, and the one check `tests/check_campaigns.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_campaigns.py`, exit 0), **one coupon redeemed to its limit and then refused, plus an out-of-window rejection**:
  - **campaign-scoped discounts ride the one engine, not a second one** — a rule scoped to campaign `XMAS` wins at 180.000000 under `XMAS` and is not even considered under `EASTER` or with no campaign named, where the any-campaign rule applies instead. The campaign rides `_order_key`, so it is a dimension of the documented resolution order rather than a parallel path.
  - **a coupon applies only within its validity window** — 2026-09-30 is refused as *not yet valid* naming the opening date, 2026-11-01 is refused as **expired** naming the closing date, and the opening and closing days are **inclusive** (redeemed on the closing day). An **unknown code** is refused naming the code it could not find, and a code filed under another company is not found either.
  - **and only up to its usage limit** — the coupon capped at one redemption is spent once, and the second attempt is refused with the limit and the count in the message (`allows 1 redemption(s) and has used 1`); the uncapped coupon beside it keeps applying, so a limit is not a global stop. Null is *no limit*, asserted apart from zero.
  - **a coupon cannot be stacked beyond the configured allowance** — the allowance is stated per coupon (the plan states none, so none is invented) and the **strictest allowance on the document governs**: with an allowance-2 coupon already standing, adding an allowance-1 coupon is refused (`the strictest allowance is 1`), while the same coupon on a fresh document applies. Redeeming the same code twice on one document is refused too.
  - **redemption is recorded against the document that used it** — the row names `document_type` and `document_id`, is asserted to be the document that spent it, and is **append-only**: a hand-written `UPDATE` and `DELETE` are both refused at COMMIT by T-0.AUDIT.01's trigger. That is what stops a spent coupon being respent.
  - **the whole path is drivable through the published API** — `POST {BASE}/coupons` files a coupon with its window, limit and allowance (201), an unknown code is refused as `422 coupon_error`, a redemption answers with the amount and the document it was recorded against, and the same coupon used after its window is refused naming the closing date.
  **Not in scope, deliberately**: marketing automation and any customer segmentation beyond the tier dimension (neither is named in the plan), and a checkout flow that applies coupons to a whole quotation — T-3.SALES.07 owns the coupon dimension and its redemption, and nothing here re-prices a document. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**51/51**) and `tests/check_ledger_integrity.py` green; the frontend's `npm run check` green on all seven after the contract was re-published and re-vendored (`ERPfrontend/contract/v1/openapi.json` byte-identical, `lib/contract.d.ts` regenerated).
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `AR` — Accounts Receivable (§2.1)

#### Task ID: T-3.AR.01
- **Title**: Customer invoicing with GL posting
- **Description**: Raise customer invoices from sales orders or standalone, with tax, currency and due dates, and post them to the GL through the shared posting interface.
- **Plan source**: §2.1 Core Components (Accounts Receivable (AR) – customer invoices), §4 Phase 3 bullet 3
- **Scope**: Invoice document and its posting. Aging is T-3.AR.02; settlement via gateway is T-3.AR.05.
- **Variables / Config**: `tax_pack`, `transaction_currency`, account mapping (receivables control, revenue).
- **Dependencies**: T-3.SALES.01, T-3.SALES.05, T-1.ACCT.03
- **Acceptance Criteria**: An invoice posts a balanced entry to the receivables control account; an invoice for goods already shipped references the shipment and does not re-issue stock; a duplicate invoice for the same order and amount is refused; tax is applied per the active pack.
- **Evidence**: **DONE** — the customer invoice, in the **backend repository**. New package `app/ar/` (`app/ar/invoices.py`: `CustomerInvoice`, `CustomerInvoiceLine`, `CustomerInvoiceSettlement` — append-only under T-0.AUDIT.01, `create_invoice`, `post_invoice`, `settled_amount`, `open_amount`, `settle`, `open_invoices`, `invoice_by_number`; the settlement's `uq_customer_settlement_once_per_source` is added by T-3.AR.05, which needs it) and the shared sales-side tax resolution `app/sales/tax.py` (`SALES_DOCUMENTS`, `active_market`, `sales_rule`, `rule_by_code`, `tax_on`), the one check `tests/check_customer_invoice.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_customer_invoice.py`, exit 0), **a posted invoice for a shipped order plus a duplicate rejection**:
  - **one balanced entry through the mapping** — the invoice from a 10-widget + freight order posts debit 1100 `339.360000` / credit 4000 `303.00` / credit 2200 `36.360000`; the accounts come from `receivables`, `revenue` and `output_tax` (T-1.ACCT.03), so no code is written into the module.
  - **tax is applied per the active pack** — the pack's `VAT-OUT-12` is resolved through `app.localization` for `sales_invoice`, and `sales_rule()` refuses a pack that taxes `sales_order`, `sales_invoice` and `pos_sale` differently (the three are the same sale at three moments). A line naming `VAT-ZERO` is charged 0 while its standard-rated sibling is charged 12, and `VAT-IN-12` — a **buying-side** rule — is refused rather than charged on a sale.
  - **goods already shipped are referenced, not re-issued** — the invoice names its order and its shipment (both checked to be one chain with the customer and the shipment line's order line), the posting touches no stock account, and the stock ledger is asserted at the same count before and after posting: the shipment's own issue is the only one.
  - **a duplicate is refused, not discovered later** — same customer, same order, same amount is refused by the module's message (`order 'SO-1' has already been invoiced for 339.360000 as 'AR-1001'`), the partial unique index `uq_customer_invoice_duplicate` refuses a hand-written one at COMMIT, and a standalone invoice (no order) is not swept into that rule.
  - **what is owed is derived from settlements** — a `100.000000` receipt leaves `239.360000` of `339.360000`; over-settling is refused with both figures; settling a draft is refused; the settlement row is **append-only** (a hand-written `UPDATE` and `DELETE` are both refused at COMMIT).
  - **foreign currency and terms** — a USD invoice keeps USD and its entry carries the rate for its own posting date (`58.5000000000`), so its base amount is derivable exactly; the due date is the invoice date plus the customer's own terms.
  - **the refusals at entry** — no lines, a blank number, a zero-quantity line and a shipment that is not the named order's are each refused by message.
  **Not in scope, deliberately**: aging (T-3.AR.02), gateway settlement (T-3.AR.05) and the order-time credit check (T-3.AR.06) — this task owns the invoice document and its posting, and every later AR task reads what it records. No HTTP endpoint: none of the four criteria names one, and T-2.AP.01's supplier invoice — this document's mirror — is service-level too. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**52/52**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.AR.02
- **Title**: AR aging report
- **Description**: Age open customer balances by due date and configurable buckets, per customer and per company.
- **Plan source**: §2.1 Core Components (AR – aging)
- **Scope**: Aging report and export.
- **Variables / Config**: `company_id`, `report_schedule`, aging buckets (not stated).
- **Dependencies**: T-3.AR.01
- **Acceptance Criteria**: Each aged amount traces to an open invoice; the total equals the receivables control account; partial receipts reduce the correct bucket; buckets are configurable and shown on the report.
- **Evidence**: **DONE** — aging the receivables, in the **backend repository**. New module `app/ar/aging.py` (`checked_buckets`, `Report` with `totals`, `totals_by_currency`, `compare_to_control`, `by_customer`, `by_currency`, `aging`, `aging_csv`) and the ledger reader it compares against `app/ar/reconciliation.py` (`control_balance` — the half T-3.AR.07's reconciliation itself lands on), the one check `tests/check_ar_aging.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ar_aging.py`, exit 0), **an aging report over a dataset with partial receipts, reconciled to the control account**:
  - **every aged amount traces to an open invoice** — six aged rows each carrying their invoice number, customer, invoice and due dates, days past due and bucket; the buckets are the report's own `DEFAULT_BUCKETS` (`current`, `1-30`, `31-60`, `61-90`, `90+`), stated on the report because the plan leaves them unstated.
  - **partial receipts reduce the correct bucket** — a `50.00` receipt against a `168.000000` invoice leaves `118.000000` open in the **1-30** bucket and nothing else moves; a fully settled invoice drops out of the population entirely.
  - **the total equals the receivables control account** — per currency, read from `journal_line` through the mapping key `receivables`, with the difference stated: PHP `1854.000000` against `1854.000000` and USD `112.000000` against `112.000000`, both zero; an injected `300.00` posted straight to the control account is reported as a difference of `-300.000000` rather than absorbed. The headline `totals` add the rows as they stand and are documented as meaningful only while the company invoices in one currency — `by_currency()` is what the comparison uses.
  - **buckets are configuration and are carried on the report** — three custom bands age the same population to `{'not due': 672, 'due now': 510, 'late': 784}`, the same total, and every set that would lose or double-count an invoice is refused: a gap, an overlap, a set starting after day 0, two open ends, a closed last bucket, an empty set and a reversed span.
  - **per customer and per currency** — the per-customer view (`ACME PHP 1574`, `BOREAL PHP 280`, `BOREAL USD 112`) adds back to the report total, and the export carries the buckets by name and the control comparison.
  **Not in scope, deliberately**: the reconciliation's own verdict and explanation (T-3.AR.07 — this task adds only the control-balance reader it shows beside its total) and dunning (T-3.AR.04), which reads the same rows. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**54/54**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.AR.03
- **Title**: Recurring billing
- **Description**: Define recurring invoice templates and schedules, and generate invoices on their cadence.
- **Plan source**: §2.1 Core Components (AR – recurring billing)
- **Scope**: Templates, schedules and generation. Collection of the resulting invoices goes through the normal AR path.
- **Variables / Config**: `recurring_billing_cycle`, `tax_pack`, `transaction_currency`.
- **Dependencies**: T-3.AR.01
- **Acceptance Criteria**: A template generates invoices exactly on its cadence with no duplicates if the job reruns; a paused template generates nothing; a generated invoice is indistinguishable from a manual one for aging and dunning purposes; the generation is idempotent and its failures are reported.
- **Evidence**: **DONE** — recurring billing, in the **backend repository**. New module `app/ar/recurring.py` (`RecurringTemplate`, `RecurringTemplateLine`, `RecurringInvoiceRun` — append-only under T-0.AUDIT.01, `period_start`, `period_key`, `create_template`, `pause_template`, `template_by_code`, `runs_for`, `periods_due`, `generate_due`, `Generation`), the one check `tests/check_recurring_billing.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_recurring_billing.py`, exit 0), **a monthly template run twice for the same period producing one invoice, plus a paused template producing none**:
  - **the cadence is the template's, not the job's** — period *n* starts one cadence after `starts_on` (`_add_months` clamps a 31st to the target month's last day rather than rolling over), so a late run cannot shift a period. The same start date gives monthly `2026-01…03`, quarterly `2026-01, 2026-04, 2026-07` and weekly `2026-W37…W40`, each keyed as its own cadence names it.
  - **no duplicates when the job reruns** — a second run over the same period raised nothing at all; the run row (`uq_recurring_run_period_once`) is the idempotency, and the invoice number carries the period (`HOSTING-2026-01`), so even a lost run row collides with T-3.AR.01's number rule. Each period's invoice and its run row are committed **together**, so a crash cannot leave an invoice nobody recorded.
  - **a paused template generates nothing** — it comes back in `skipped` with the reason (`the schedule is paused`); resumed, it billed the six periods it had missed, in order, each dated its own period's start.
  - **a generated invoice is indistinguishable from a manual one** — each is an ordinary posted `CustomerInvoice` with a balanced entry through the mapping, the customer's 30-day terms, and the aging report (T-3.AR.02) ages all twelve rows to a `0.000000` difference against the receivables control account.
  - **failures are reported, and retried** — a template whose line names a buying-side classification (`VAT-IN-12`) is refused by `app.sales.tax` for each of its seven reached periods; every refusal is reported with its reason and its period, no run row is written for it, the next run retries all seven, and the billable template beside it was billed in the same run.
  - **the run record is history** — the database refuses a hand-written second run row for a period and refuses to edit the one it holds.
  - **the refusals at entry** — an unknown cadence, an empty template, a duplicate code, an end date before the start and a zero-quantity line are each refused by message.
  **Not in scope, deliberately**: collecting the resulting invoices — they go through the normal AR path (T-3.AR.05's settlement and T-3.AR.04's dunning read them like any other invoice) — and any scheduling of the job itself, which is the runner's business, not this module's. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**54/54**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.AR.04
- **Title**: Dunning — reminder levels, escalation and delivery
- **Description**: Define dunning levels by days past due with their templates, generate reminders for overdue invoices, and deliver them through the integration boundary by email/SMS.
- **Plan source**: §2.1 Core Components (AR – dunning), §4 Phase 3 bullet 3, §3 row "Integrations"
- **Scope**: Dunning levels, generation and delivery. Interest/penalty charging is not named in the plan and is not built here.
- **Variables / Config**: `dunning_levels`, `dunning_channel`, `report_schedule` cadence for reminder runs.
- **Dependencies**: T-3.AR.02, T-0.INT.01
- **Acceptance Criteria**: Each overdue invoice lands in exactly one level per run according to its days past due; an escalated invoice does not also receive the earlier level's reminder; delivery attempts and failures are visible in the delivery log; a settled invoice receives no further reminders; running the dunning job twice for a period does not duplicate reminders.
- **Evidence**: **DONE** — dunning, in the **backend repository**. New module `app/ar/dunning.py` (`DunningLevel`, `DunningReminder` — append-only under T-0.AUDIT.01, `checked_levels`, `define_level`, `levels`, `level_for`, `run_dunning`, `DunningRun`, `reminders_for`), the one check `tests/check_dunning.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_dunning.py`, exit 0), **one invoice escalated through two levels and one settled mid-way, with the delivery log and the duplicate-run check**:
  - **exactly one level per run, chosen by the days past due** — a 10-day-late invoice landed in `SOFT` (email) and a 55-day-late one in `FINAL` (sms); a billed-but-not-yet-due invoice reached none and is reported as skipped rather than silently dropped. The schedule is refused unless it partitions the days it covers (a gap, an overlap, a level starting before day 0, two open ends, a closed last level, an empty schedule, an unknown channel).
  - **an escalated invoice does not receive the earlier level's reminder** — 31 days later `AR-EARLY` escalated to `FINAL` and was sent that level **alone**; its history reads `['SOFT', 'FINAL']` with one reminder per level per period. The unique `uq_dunning_reminder_once_per_run` on (invoice, level, run) is what makes it hold however often the period is re-run, and the database refuses a hand-written duplicate.
  - **delivery is T-0.INT.01's, and its failures are visible** — every reminder is sent through `send_outbound`, so the delivery log carries the attempts and the outcome: two `sent` rows for the first run, and a channel that refuses an address leaves `Refused: mailbox unavailable` as the log's `last_error` **and** the reason on the reminder. A customer with no address on the level's channel is recorded with that reason (`no email address on the customer's primary contact`) and no delivery, rather than silently not reminded.
  - **a settled invoice receives no further reminders** — it is skipped (reported in `DunningRun.settled`) and has no reminder row at all, even though its due date is long past; and it is still skipped by the later run.
  - **running the job twice for a period duplicates nothing** — the second run wrote no reminder at all and reported both open invoices as already reminded.
  - **a reminder is history** — the row cannot be edited or removed (append-only); the outcome is known before it is written, so nothing here is ever updated.
  **Not in scope, deliberately**: interest and penalty charging (the plan does not name it) and any scheduling of the run itself. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**58/58**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.AR.05
- **Title**: Payment gateway webhooks and settlement posting
- **Description**: Receive payment-gateway webhooks, match them to open invoices, and post the settlement including any gateway fee, handling partial and failed payments.
- **Plan source**: §2.1 Core Components (AR – payment gateway webhooks), §4 Phase 3 bullet 3, §3 row "Integrations"
- **Scope**: Inbound payment events and their posting. Gateway-specific configuration uses the T-0.INT.01 boundary; refund/reversal of a captured payment is included only if the gateway event exists.
- **Variables / Config**: `integration_endpoints`, `transaction_currency`, `company_id`.
- **Dependencies**: T-3.AR.01, T-0.INT.01
- **Acceptance Criteria**: A duplicated webhook settles the invoice once (idempotency); an unmatched payment is parked and reported rather than silently dropped; a partial payment leaves the correct open amount; a failed payment leaves the invoice open and is recorded; the settlement posts a balanced entry.
- **Evidence**: **DONE** — gateway webhooks and their settlement, in the **backend repository**. New module `app/ar/gateway.py` (`GatewayPayment`, `record_payment`, `post_settlement`, `match`, `parked_payments`, `payment_report`, `payments_for`), the one check `tests/check_gateway_payments.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_gateway_payments.py`, exit 0), **a duplicated webhook, a partial payment and an unmatched payment, each with the resulting invoice state and posting**:
  - **a duplicated webhook settles once** — the replayed delivery is answered from T-0.INT.01's `inbound_event` record without the handler running again (`repeated=True`), and a **second event key** describing the same payment (`payment.created`, then `payment.succeeded`) is answered from the `gateway_payment` row, which is unique on the gateway's own payment reference. The invoice ends with one settlement and one entry from two recorded deliveries. `customer_invoice_settlement` gained `uq_customer_settlement_once_per_source` (one document settles one invoice once) as this task's second guard, and the database refuses a hand-written second row.
  - **the settlement posts, fee apart** — bank debited `1100.000000`, the gateway's `20.00` fee debited to the company's `payment_fees` account, receivables credited the whole `1120.000000`; the entry balances, is stated in the invoice's currency, and carries `source_type='gateway_payment'`.
  - **a partial payment leaves the correct open amount** — `100.00` against a `224.000000` invoice leaves `124.000000` open and is recorded as `partial`.
  - **a failed payment leaves the invoice open** — the gateway's failure is recorded with its reason (`insufficient funds`), the invoice stays at `336.000000` open, and no settlement and no entry are written.
  - **an unmatched payment is parked and reported** — it names an invoice that does not exist, is parked with the reason, appears in `parked_payments`, and is placeable when the invoice exists: matching it settles it and empties the parked list. A payment in another currency, and one larger than what is open, are each parked with **both figures** in the reason, and the invoice is untouched.
  - **malformed events are refused at entry** — an unclassified outcome, no reference, a fee larger than the payment and a float amount are each refused **before** the boundary records anything, so the delivery count is unchanged and no receivable is settled.
  - **whose money it is, is recorded with it** — `gateway_payment` carries `customer_id`, taken from the event's optional `customer` code or from the customer the **named invoice** belongs to. A payment parked for another reason (another currency, more than is open) still knows whose money it is; a receipt that names a customer nobody has — or a party that is not a customer — is parked with that reason rather than attributed to a guess; and a payment attributed to one customer while naming **another's** invoice is parked for that instead of settling an account its money never came from. The figure T-3.AR.06 reads is therefore either right or absent, never somebody else's.
  - **the report accounts for every payment** — counts by state (`settled` 2, `partial` 1, `parked` 6, `failed` 1 — the two parked on an invoice, plus the on-account receipt, the two nobody known paid, and the mismatch), `1276.000000` collected, and the seven unplaced payments spelled out with their reasons.
  **Not in scope, deliberately**: refunds and reversals, which the plan names only "if the gateway event exists" and no gateway here states one; and dunning (T-3.AR.04), which reads the same invoices. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**56/56**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: L
- **Owner Role**: Integration Engineer
- **Status**: DONE

#### Task ID: T-3.AR.06
- **Title**: Credit limit enforcement and exposure calculation
- **Description**: Compute a customer's live exposure across open invoices, unbilled orders (including shipped quantities not yet invoiced) and payments, and enforce the limit on new commitments per the configured mode.
- **Plan source**: §2.1 Core Components (AR – credit limits), §4 Phase 3 bullet 1
- **Scope**: Exposure and enforcement. The order-time check lives in T-3.SALES.04 and must reuse this calculation.
- **Variables / Config**: `credit_limit`, `credit_check_mode`, `transaction_currency`.
- **Dependencies**: T-3.SALES.01, T-3.AR.01
- **Acceptance Criteria**: Exposure includes open invoices, all unbilled order quantities (including shipped quantities not yet invoiced) and on-account receipts correctly, and the components are itemised; the same exposure figure is used by the order-time check (one implementation); an increase in exposure from a new invoice is immediately visible to the next order check; a limit change is audited.
- **Evidence**: **DONE** — the backend implementation and the check `tests/check_credit_exposure.py` (eleven sections), in the **backend repository**. The review that reopened this id found that the recorded check did not establish exposure during the ship-before-invoice interval; the interval is now proved end to end, including its extreme and the control it exists for:
  - **the components are itemised** — invoiced `560.000000`, received `60.000000`, unbilled orders `1000.000000` and on-account receipts `50.000000` → `1450.000000`, with `open_invoices` derived (invoiced less received) and the document-level backing available through `open_items` (`AR-1 500.000000`, `AR-3 112.000000`).
  - **review finding, resolved — shipped, uninvoiced quantities are preserved** — the order is measured as its lines less what has been **invoiced** for it (`_billed_net`), never less what has *shipped*, so the commitment survives the gap between T-3.SALES.05 raising `shipped_quantity` and T-3.AR.01 raising the invoice. The regression the finding asked for is section 11: **SO-5 ships all ten units with nothing invoiced** and the exposure still states `1000.000000`, the order-time check then refuses the next order on `1100.000000` over the `1000` limit (the figure that dropping the shipped goods would give is `100`), and invoicing moves `1000.000000` out of `unbilled` and `1120.000000` into `invoiced` **once, not twice** — the total is the invoice's gross, not the order *and* the invoice. Section 10 proves the partial case (four of ten shipped left the `1700.000000` unbilled where it was). Proved red on the old arithmetic: the whole-shipment assert fails with `unbilled_orders: 0.000000`, so the check exercises the repair rather than agreeing with it.
  - **one implementation** — confirming an order with nothing stated is judged against the statement's own `1450.000000`: the refusal reads *"would take the customer to 1550.000000, over the agreed limit 1000.000000"* (the order's own 100.00, untaxed, as `order_total` defines it). `confirm_order` no longer requires a caller-stated number, and a number that *is* stated is still validated and recorded verbatim (`700.000000`).
  - **an increase is immediately visible** — a new invoice took the exposure to `1562.000000` and the very next order check blocked on `1662.000000`, with nothing recomputed by hand.
  - **on-account receipts reduce it, and a credit is never a negative exposure** — money received with no invoice to apply it to (T-3.AR.05's parked payment, attributed to the customer the event names) is subtracted; a `2000.00` receipt took the total to `0.000000` with `438.000000` held as a named credit, which is `CreditDecision`'s own non-negative rule respected rather than broken.
  - **a receipt nobody has attributed credits nobody** — the on-account bucket is read **per customer** (`gateway_payment.customer_id`), so a `900.00` receipt naming no customer left the customer that asked *and* the second customer with an open invoice exactly where they were, while the money stayed on T-3.AR.05's parked list. Crediting it to every customer would have let one customer's order pass on another customer's prepayment — the one way this control could be defeated from inside.
  - **a limit change is audited** — `set_credit_limit` writes the `customer` row and T-0.AUDIT.02's trail records it: the new trail row names the before (`1000.0`) and the after (`2500.0`). No new mechanism was needed; the check asserts the existing one.
  - **beside the limit** — `exposure_against_limit` states the figure, the limit, the headroom and whether it is breached, and a customer with **no limit agreed** reads `null` rather than a zero ceiling.
  - **per currency** — a USD invoice is reported in `other_currencies` and never added into the PHP figure; asked for in USD, the same function states that currency's total with PHP beside it.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**65/65**) and `tests/check_ledger_integrity.py` green (its own two injections reported red first). This id touches no contract and no frontend file, so the frontend pipeline has nothing of its to check.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.AR.07
- **Title**: AR to GL reconciliation and receivables control check
- **Description**: Prove that open customer balances equal the receivables control account per company and period, and report differences.
- **Plan source**: §3 row "Double-Entry Integrity"
- **Scope**: The reconciliation only.
- **Variables / Config**: `company_id`, `base_currency`.
- **Dependencies**: T-3.AR.01, T-3.AR.05
- **Acceptance Criteria**: Subledger open balance equals the control account to currency precision; an injected mismatch is reported; gateway settlements and partial receipts are reflected correctly.
- **Evidence**: **DONE** — the receivables control check, in the **backend repository**. `app/ar/reconciliation.py` completed (`subledger_balance`, `currencies_in_use`, `reconcile`, `explain`, alongside T-3.AR.02's `control_balance`), the one check `tests/check_ar_reconciliation.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_ar_reconciliation.py`, exit 0), **a clean reconciliation plus a reported mismatch case**:
  - **the subledger equals the control account to currency precision** — `1344.000000` against `1344.000000`, difference `0.000000`, per currency; the subledger side is each invoice's derived `open_amount` and the control side is the `journal_line` rows the postings wrote at the account the `receivables` mapping points at, so the two share nothing but the postings.
  - **partial receipts are reflected correctly** — a `400.00` receipt moved both sides to `944.000000`, because the subledger side *is* the settlement that was appended rather than a figure kept beside it.
  - **gateway settlements are too, fee and all** — a `150.00` settlement whose `7.50` fee was debited to the fee account left both sides at `794.000000`: the fee never touches receivables, so it cannot drag the comparison apart.
  - **an injected mismatch is reported, never absorbed** — a `250.00` posting made straight to the control account (nothing behind it in the subledger) is reported as a difference of `-250.000000`, with both figures and the gap stated; nothing in the module corrects it.
  - **per currency** — a USD invoice reconciles in USD (`560.000000` = `560.000000`) and leaves the PHP difference untouched; the currencies swept are those present in the subledger **or** the control account, so a control-only posting cannot hide by having no invoice.
  - **a period reconciles as a period** — with `start`, **both** sides are that window's movements: invoices raised less settlements posted inside it (`subledger_movement`) against the account's own movement, so a window that does not open at the first posting is compared with the same measure on both sides instead of a position against a movement — which would have reported every correct period as a difference. The check asserts a window in which nothing moved balances at zero on both sides while the position to date is still outstanding, and that the injected `250.000000` is the difference reported against the window's own movement, the same figure the position-to-date comparison states.
  **Not in scope, deliberately**: correcting anything it finds (that is a finding for whoever caused it) and dunning, which reads the same invoices. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**58/58**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: S
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `POS` — Point of Sale (§2.5)

#### Task ID: T-3.POS.01
- **Title**: POS sale transaction with barcode scanning and receipt (online)
- **Description**: Ring up a sale by scanning item barcodes, apply the pricing engine, take payment, produce a receipt, and post the sale and its stock movement.
- **Plan source**: §2.5 Core Components (POS (offline-capable, barcode, cash drawer, Z-Reports)), §4 Phase 3 bullet 5 (POS module (online first, offline later)), §6 metric 6
- **Scope**: **Online only** — offline operation is Phase 6 (T-6.OFFLINE.01). Runs in the frontend container against the API, with no direct database access (T-0.API.02). No loyalty or customer-display features (not named).
- **Variables / Config**: `pos_mode` (`online`), `barcode_symbology`, `pricing_rule_dimensions`, `payment_tender_types`, `pos_latency_budget`.
- **Dependencies**: T-1.INV.01, T-3.SALES.06, T-1.ACCT.03
- **Acceptance Criteria**: Scanning a barcode resolves to the item/variant and adds the correct line at the engine-resolved price; the sale posts a balanced entry and issues stock from the POS location; the receipt is reproducible from the stored sale; an unknown barcode is rejected rather than sold at zero; a sale cannot be completed without a settled tender.
- **Evidence**: **DONE** — the POS sale, in the **backend repository** (the till's client is in the **frontend repository**, `app/pos/` + `lib/pos.ts` + its test, against the published contract). New package `app/pos/` (`app/pos/sales.py`: `PosSale`, `PosSaleLine`, `PosTender`, `open_sale`, `scan`, `tender`, `complete_sale`, `receipt`, `sale_by_number`), `app/stock/gl_posting.py`'s `SOURCE_ACCOUNT_KEYS` given the `pos_sale` source (a till's issue is a sale, so it posts to the same cost-of-sales key a fulfilment issue does), the API endpoints `POST {BASE}/pos/sales`, `.../scan`, `.../tender`, `.../complete`, `GET .../receipt` and `POST .../void`, the republished `contract/v1/openapi.json` re-vendored byte-identically into the frontend, and the one check `tests/check_pos_sale.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_pos_sale.py`, exit 0), **a multi-line sale with its stock ledger entries, posting and reproducible receipt, plus an unknown-barcode rejection**:
  - **a scan resolves and the engine prices it** — the barcode resolves through T-1.INV.01's `item_from_barcode` to the item, and to a **variant** where the code names one; the line takes T-3.SALES.06's resolved price (50.00 → `45.000000` under the `GOLD-10` tier rule, priority recorded) and the pack's `VAT-OUT-12`, so a receipt can say why the price is what it is.
  - **completing posts one balanced entry** — cash debited `181.440000`, revenue credited `162.000000`, output tax `19.440000`; every account comes from the `cash`/`bank`/`revenue`/`output_tax` mappings, and the tender's own account is chosen by its type.
  - **completing issues the stock** — both lines issued out of `TILL-1` through T-1.INV.05's `issue`, which values them and writes the stock ledger: two movements against the sale, Widget 500 → 498, each line naming its movement.
  - **the receipt is reproducible** — the same figures come back after a second, stronger pricing rule is added, because the receipt is read from the stored sale rather than re-priced.
  - **nothing moves until the sale is complete** — an open basket has no movement and no posting, a completion whose tenders do not cover it is refused with what is owed, an unknown barcode is refused rather than sold at zero, and a card that would over-pay is refused (change comes out of the drawer).
  - **the refusals at entry** — no lines, a blank number, a duplicate number and a zero-quantity scan are each refused, and so is a tender of `ten pesos` (a figure the boundary cannot read is the platform's `422 pos_error`, never a 500) and a sale **priced at nothing**: `complete_sale` refuses it by name before the stock moves, where the posting it would have made is a single zero line — not an entry at all — and the sale stays open.
  - **the till's own reading of a sale** (the **frontend repository**, `lib/pos.ts` + `tests/pos.test.ts`) — amounts are added exactly at the money scale and a tender whose amount cannot be read is counted as **invalid** rather than as zero, so a sale with one unreadable tender is never shown as settled; a basket **worth nothing is not settled either** (`0 >= 0` showed an empty till roll as paid, where the domain refuses to complete a sale worth nothing); the **void/refund control stays on the screen once a sale completes**, since refunding a completed sale is exactly what the backend's own `void` op does; every control carries a visible label and a step's answer is announced (`role="status"`, `alert` for a refusal); and the till is **keyed on its terminal**, so changing terminal in the query drops the basket rather than letting TILL-1's open sale be completed against TILL-2's shift and location.
  **Not in scope, deliberately**: offline operation (T-6.OFFLINE.01), loyalty and customer-display features (the plan names neither), and the Z-Report (T-3.POS.04). **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**65/65**) and `tests/check_ledger_integrity.py` green; the frontend's `npm run check` green after the contract was re-vendored.
- **Estimated Effort**: L
- **Owner Role**: Full-stack Engineer
- **Status**: DONE

#### Task ID: T-3.POS.02
- **Title**: Cash drawer and payment tendering (cash, card, gateway, split)
- **Description**: Support the tenders a POS till accepts, including split payments across tenders, change calculation and drawer movements (paid in/out).
- **Plan source**: §2.5 Core Components (POS (… cash drawer …))
- **Scope**: Tendering and drawer movements. Shift open/close and cash count are T-3.POS.03.
- **Variables / Config**: `payment_tender_types`, `transaction_currency`.
- **Dependencies**: T-3.POS.01
- **Acceptance Criteria**: A sale settles only when the tendered total covers it; change is computed correctly and cannot go negative; a split payment records each tender separately with its own settlement path; drawer movements outside a sale are recorded with a reason and appear in the shift totals.
- **Evidence**: **DONE** — tendering and the drawer, in the **backend repository** (the drawer's screen is in the **frontend repository**). New module `app/pos/drawer.py` (`DrawerMovement`, `record_movement`, `paid_in`, `paid_out`, `movements_for`, `movement_total`, `cash_sales_total`, `change_paid`, `tender_breakdown`, `drawer_state`), `app/pos/sales.py`'s tendering completed (`PosTender` gaining `tender_no` so a receipt lists its tenders in the order the till took them), the API endpoint `POST {BASE}/pos/drawer-movements` and the contract re-vendored, and the one check `tests/check_pos_tender.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_pos_tender.py`, exit 0), **one cash sale with change, one split cash/card sale, and one paid-out, all reflected in the shift's totals**:
  - **a sale settles only when the tenders cover it** — an under-paid sale is refused with what it is owed, and a card tender that would over-pay is refused (`change comes out of the drawer`), because a card is read for the amount it is charged.
  - **change is computed and cannot go negative** — the 120.00 cash sale took `112.000000` and gave `8.000000` back; `tendered` and `applied` are stored apart, with `applied <= tendered` in the schema, so a sale's change is non-negative by construction rather than by a check.
  - **a split payment records each tender separately with its own settlement path** — cash `50.000000` to the drawer's account and the card `62.000000` to the bank's, each its own line in one balanced entry, with the card's authorisation on its own tender row.
  - **drawer movements outside a sale are recorded with a reason and move the drawer** — a `200.00` paid-out (window cleaner) and a `500.00` paid-in (opening float) leave `drawer_state`'s expected cash at the figure the documents give, and both name who made them.
  - **the refusals** — a movement with no reason, with no actor, or a non-positive amount is each refused: cash that left the drawer unexplained cannot be counted by anybody. An amount nobody can read (`zzz`, and a `NaN` or an `Infinity`) is refused **by name** from the service and as `422 drawer_error` over the API rather than raised further in as a `decimal.InvalidOperation` the boundary would answer 500 to; a figure the till mistyped is the till's mistake, not the platform's fault.
  - **the movement knows its shift** — `pos_drawer_movement` carries `shift_id`, taken from the shift trading on that terminal when the cash moved, so T-3.POS.03's `shift_totals` reads a shift's movements as *its own* rather than every movement on that till that day.
  - **the breakdown keeps the paths apart** — `tender_breakdown` states what each tender type took and covered, rather than one lumped number.
  **Not in scope, deliberately**: shift open/close and the cash count (T-3.POS.03), which reads `drawer_state`; and any ledger posting for a movement (the money was already the company's — an expense has its own document). **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**65/65**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Full-stack Engineer
- **Status**: DONE

#### Task ID: T-3.POS.03
- **Title**: Retail shift management
- **Description**: Open and close a till shift per store/terminal, with opening float, closing count, expected-versus-counted variance and the shift's document list.
- **Plan source**: §2.5 Key Capabilities (Retail shift management)
- **Scope**: Shift lifecycle and reconciliation of a shift. Z-Reports (the printed/exported close summary) are T-3.POS.04.
- **Variables / Config**: `cash_drawer_required`, `shift_definitions`, `payment_tender_types`.
- **Dependencies**: T-3.POS.02
- **Acceptance Criteria**: A shift cannot be opened twice on one terminal; sales cannot be rung up outside an open shift; closing records expected versus counted cash with the variance, and requires a reason when the variance is non-zero; a closed shift cannot accept new sales.
- **Evidence**: **DONE** — retail shift management, in the **backend repository** (the shift screen is in the **frontend repository**). New module `app/pos/shifts.py` (`PosShift` with the partial unique index `uq_pos_shift_open_terminal`, `current_shift`, `open_shift` — which claims the drawer's own unattributed movements of that day — `shift_sales`, `shift_movements`, `shift_totals`, `close_shift`, `closed_shifts`, `require_open_shift`), the company policy the task names (`app/company.py`: `cash_drawer_required`, `cash_drawer_required_for`, `set_cash_drawer_required`), `app/pos/sales.py` gaining the shift on each sale (stamped when a basket is rung up and again at completion, so a sale belongs to the drawer that took its money), the API endpoints `POST {BASE}/pos/shifts`, `GET .../shifts/current`, `POST .../shifts/{id}/close` and `POST {BASE}/pos/cash-drawer-policy`, the contract re-vendored, and the one check `tests/check_pos_shifts.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_pos_shifts.py`, exit 0), **one shift opened, traded and closed, with a non-zero variance and its reason**:
  - **a shift cannot be opened twice on one terminal** — the service refuses it and the **partial unique index** refuses a hand-written second row, which is what makes the rule a fact of the schema.
  - **sales cannot be rung up outside an open shift** *when the company asks for one* — `cash_drawer_required` is the task's own variable; unstated reads as "no drawer management" (the ordinary shop sells without a shift, asserted), and stating it refuses a completion on a terminal with no shift **before** any stock leaves the shelf.
  - **closing records expected versus counted** — the expected figure is derived (T-3.POS.02's `drawer_state` plus the shift's float: `500.00` float + the cash sale's net `112.00` − `60.00` paid out = `552.000000`), the count is what was found, and the variance between them is stored with the shift.
  - **a non-zero variance needs a reason** — it is refused without one (`the drawer counted 600.00 against an expected 552.000000`), the reason is recorded, and a shift that balanced closed with a zero variance and no reason asked for.
  - **a closed shift takes no more sales** — a sale rung after the close finds no open shift on that terminal.
  - **the totals are derived every time** — sales, net, tax, gross, the tenders kept apart, the movements and the change paid are added from the documents, and the opening float is counted **once**: the drawer expectation adds the shift's own `opening_float` and leaves out a **paid-in** movement that carries the float's reason (it is that figure, not money beside it), while anything the drawer **paid out** counts whatever words its reason uses — a `5.00` withdrawal somebody described as an "opening float" is money that left the till and takes the expectation from `90.000000` to `85.000000`, which matching on the reason text alone had silently dropped.
  - **the movements are the shift's own** — each movement is stamped with the shift trading when the cash moved (and a shift opening claims the drawer's movements made on that terminal that day before it opened, which is the float case above). The day's **second shift on one till** therefore opens on its own float and closes on it with no variance, where reading movements by terminal-and-day gave it the first shift's `10.00` paid-out and refused its correct count for a variance nobody made.
  - **a closed shift is not restated** — a basket opened inside a shift and completed *after* it closed carries no shift at all when the company asks for no drawer, and the closed shift's own report reads exactly what it did before; with the policy on, the completion is refused instead. Either way a signed-off Z-Report cannot grow a sale after the fact.
  - **the policy is readable over the API** — the `cash_drawer_required` a till trades under comes back on `GET {BASE}/companies/current` (the read endpoint was the one place still answering `null` for it), so the till's screen states the policy the drawer is actually judged by, and withdrawing it reads back as *unstated* rather than as a different answer.
  - **the drawer panel names its controls** (the **frontend repository**) — the movement-type select carries a visible label bound to it rather than relying on its value being obvious to a screen reader, and the panel reads every figure from the API's own statement instead of recomputing one.
  **Not in scope, deliberately**: the Z-Report (T-3.POS.04, which reads these totals) and any optimisation of the till's path (T-3.POS.06 measures it). **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**65/65**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Full-stack Engineer
- **Status**: DONE

#### Task ID: T-3.POS.04
- **Title**: Z-Reports (shift and day close)
- **Description**: Produce the shift/day close report: sales by tender, tax, discounts, voids, drawer variance and totals, per terminal, shift and day.
- **Plan source**: §2.5 Core Components (POS (… Z-Reports))
- **Scope**: The close reports and their export. The reporting framework delivery mechanism is T-0.REPORT.01.
- **Variables / Config**: `report_schedule`, `company_id`, `payment_tender_types`.
- **Dependencies**: T-3.POS.03
- **Acceptance Criteria**: The Z-Report totals tie to the shift's constituent sales with no rounding gap; tax is broken out per the active pack; voids and refunds appear separately; the day report aggregates the shift reports exactly (equal to the sum, not approximately) under a defined cross-midnight ownership rule; the report is immutable once the shift is closed, including after a refund of one of its sales.
- **Evidence**: **DONE** — the backend implementation and the check `tests/check_pos_zreport.py` (ten sections), in the **backend repository**. The review that reopened this id found the immutability and day-to-shift reconciliation criteria unproven; both are now established, each by the regression the finding asked for:
  - **the totals tie with no rounding gap** — `200.000000` net + `24.000000` tax = `224.000000` gross, and the tenders applied add up to exactly that; the report states the comparison (`ties`) rather than asserting it quietly.
  - **tax is broken out per the active pack** — each line's tax is the shared selling rule's, and the report sums the lines rather than re-deriving a document figure.
  - **voids and refunds appear separately** — an abandoned basket (nothing ever moved) and a refunded sale (money given back) are counted and valued as their own lines, each naming its sale, and neither is counted among the sales.
  - **a refund reverses what the sale did** — the goods go back on the shelf at the value they left at (the schema's stock ledger shows the return at `40.000000`, the issue's own value) and a **reversing entry** is posted that is the sale's own mirrored, account by account; the ledger stays append-only.
  - **a void needs a reason and an actor**, and a sale already void cannot be voided twice (it would put the goods back and reverse the revenue twice).
  - **a refund is dated when it happens, and the day it happens is the day that absorbs it** — `void_sale` takes the refund's own date (today by default) and posts the reversal there, and the closed shift's report is unchanged by it (section 10, above).
  - **the money coming back out is counted on the day it went out** — `refunds_on` reads refunds by the reversal entry's posting date, so the refund appears in its own day's report and, when it was cash, in the drawer expectation of the shift that handed it back (`refund_cash`): the check refunds `POS-K1` two days after the sale and that day states the `112.000000` given back with its drawer expecting `88.000000`. A card refund moves no cash and changes no drawer; section 10 covers the effect of a post-close refund on the original shift report.
  - **review finding, resolved — the day report *is* the sum of the shift reports** — `day_report` no longer recomputes a shift's buckets. It now takes the day's shifts — the ones that **opened** that day or **traded** on it (sold, moved cash) — and builds each line from `shift_report(shift, on=day)` itself, the shift's own report restricted to that day, so the equality is arithmetic over one function instead of two implementations kept agreeing by hand. The cross-midnight rule is stated in the docstring: a till left trading past midnight is reported on the day it traded, in its own shift's line, and its slices add back to its whole report (the opening float rides on the day the shift opened, so it is stated once). The regression is section 6 — every day line compared field by field against the actual `shift_report()` of the shift it belongs to — and section 9: the till left trading past midnight reads `112.000000` on the next day **in T3's own line**, with `0.000000` in the shiftless bucket it used to be moved into, `shift_report(T3, on=day) + shift_report(T3, on=tomorrow) == shift_report(T3)`, and the earlier day's report unchanged. Proved red on the old rule: the same check reported `next_day["shifts"] == []` and `shiftless["sales"] == 1`, which is exactly the bucket the finding objected to.
  - **review finding, resolved — a post-close refund leaves the closed shift's report alone** — `shift_sales` reads a shift's sales by **the entry they posted**, not by `status`, so a sale refunded after the close is still a sale the shift took: the money went through that till and was counted in it, and refunding it is a document of its own on its own day. The regression is section 10: the original shift's report is captured after its close, `POS-K1` is refunded **two days later** through a different shift, and `shift_report(shift)` is asserted byte for byte what it signed off — while the refunding day states the `112.000000` that went back and its own shift's drawer expects `88.000000` (the `200.00` float less the cash handed over). Proved red by making `shift_sales` skip a refunded sale (`status != 'void'`) — the mutation the finding describes: the shift's own net falls from `300.000000` to `200.000000` and the check fails, so the report is exercised rather than merely described.
  - the report carries the drawer's own count, expected figure, variance and reason.
  **Not in scope, deliberately**: interest or penalty charges. **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**65/65**), including the five other POS checks it could have moved (`check_pos_shifts`, `check_pos_sale`, `check_pos_tender`, `check_pos_reconciliation`, `check_pos_latency`) and `check_phase3_exit`, and `tests/check_ledger_integrity.py` green (its own two injections reported red first). The published contract is unchanged — the report's payload is a free-form object and only keys inside it were added — so the frontend has nothing of this id's to check.
- **Estimated Effort**: M
- **Owner Role**: Full-stack Engineer
- **Status**: DONE

#### Task ID: T-3.POS.05
- **Title**: POS to GL and stock reconciliation
- **Description**: Reconcile POS sales, tenders and stock movements against the GL and the stock ledger, exposing any difference.
- **Plan source**: §3 row "Double-Entry Integrity", §2.2 Real-time stock valuation principle
- **Scope**: The reconciliation only.
- **Variables / Config**: `company_id`, `report_schedule`.
- **Dependencies**: T-3.POS.04
- **Acceptance Criteria**: POS daily totals equal the corresponding GL revenue, tax and tender postings; POS stock movements reconcile to the stock ledger; a difference is reported per day and terminal rather than aggregated away; the reconciliation is re-runnable.
- **Evidence**: **DONE** — the POS to GL and stock reconciliation, in the **backend repository**. New module `app/pos/reconciliation.py` (`reconcile_gl`, `reconcile_stock`, `reconcile`, `per_terminal`), the API endpoint `GET {BASE}/pos/reconciliation?on=&terminal=`, the contract re-vendored, and the one check `tests/check_pos_reconciliation.py` — run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_pos_reconciliation.py`, exit 0), **a clean reconciliation for a traded day plus a reported injected difference**:
  - **POS daily totals equal the corresponding GL postings** — the day's `448.000000` gross across three sales, with revenue and output tax credited and each tender's account debited, matches the sales' own `journal_line` rows to the last decimal; the POS side is the documents and the GL side is the postings, so a missing entry shows as a difference of the whole amount.
  - **POS stock movements reconcile to the stock ledger** — every movement the day's sales carry, read from the ledger by source type and sale id **and by its own posting date**, matches the quantities their lines say left the shelf, per item and variant. Reading them by sale id alone dragged an earlier day's issue into a day that had refunded it, where no document of that day explains it.
  - **a difference is reported per day and terminal, never aggregated away** — the same figures are stated per terminal, so a day that balances while one till is short cannot hide; a till whose only document of the day is a refund has its day stated too, which is exactly the till a whole-day figure would have hidden.
  - **an injected difference is reported** — an extra `50.00` posting against a sale shows as `-50.000000` on revenue and `+50.000000` on the account it touched, a movement with no line to explain it shows on the stock side, and a tender booked to the wrong account shows on the account it should have reached.
  - **a refund is reconciled on the day it happened, beside that day's sales** — the reversal's revenue and tax are expected with the opposite sign, each tender's account carries what was handed back, and the goods returned are expected back on the shelf. The check refunds a sale rung up the day before: yesterday still states its `112.000000` of one sale, balanced, with its stock expectation at `-1`, while today states its two refunds (`224.000000` gross) beside its four sales and agrees with the ledger on revenue, cash, bank and the goods. Counting a refunded sale out of its own day, as the first draft did, left the reversal entry in the ledger with nothing on the document side to explain it, so a day with a refund could never balance.
  - **two names for one account are not a difference** — the expectations accumulate per account rather than in a literal keyed by one mapping name, so a company that banks its card takings into the account the drawer posts to is not reported short.
  - **the reconciliation is re-runnable** — the same call returns the same figures, because both sides are read from the rows each time.
  **Not in scope, deliberately**: correcting anything it finds (that is a finding for whoever caused it) and the latency measurement (T-3.POS.06). **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**65/65**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-3.POS.06
- **Title**: POS online transaction latency verification (< 2 s)
- **Description**: Measure and verify online POS transaction latency against §6's < 2 s target, and record the measured figures that the Phase 3 gate depends on.
- **Plan source**: §6 metric 6, §4 Phase 3 exit criteria
- **Scope**: Measurement and verification only. Fixing a failure is a new finding for the owner of the offending task; no optimisation work is bundled here.
- **Variables / Config**: `pos_latency_budget` (< 2 s), `pos_mode` (`online`).
- **Dependencies**: T-3.POS.05
- **Acceptance Criteria**: Latency is measured on a realistic dataset for a complete sale (scan → price → tender → post → receipt); the measured distribution is reported (not a single hand-picked run) and compared to the < 2 s budget; the measurement is repeatable in CI or a scheduled run.
- **Evidence**: **DONE** — the latency verification, in the **backend repository** (measurement only: no product code changed). The one check `tests/check_pos_latency.py`, run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_pos_latency.py`, exit 0):
  - **a realistic dataset** — a forty-item catalogue with barcodes, 1,000 units of each received into the till, a customer with a tier and four pricing rules across the tier, quantity and item dimensions, so the engine has rules to order and the issue has stock to value rather than an empty database.
  - **a complete sale, timed end to end** — scan (barcode → item → engine price → pack tax) → tender → complete (stock issue and balanced posting) → receipt, each sale committed on its own, which is what a till does.
  - **the distribution, not one run** — pass 1 over 60 complete sales: min `51 ms`, median `54 ms`, **p95 `58 ms`**, max `76 ms`; pass 2 on the same database: min `51 ms`, median `55 ms`, p95 `58 ms`, max `61 ms`. Both are printed, and the two-pass shape is what makes a regression visible as a number. Re-run after the percentile was corrected to the nearest-rank rule, on the same shape of dataset: min `70 ms`, median `91 ms`, p95 `109 ms`, max `123 ms` and min `59 ms`, median `91 ms`, p95 `103 ms`, max `118 ms` — a different machine load, the same budget, and the figure the report prints is now the rank it names.
  - **compared with the budget on the tail** — §6 metric 6's < 2 s is asserted against the **95th percentile** of both passes, not against an average, because a single slow sale is what a customer experiences. The percentile is the nearest-rank one, `ceil(fraction × n)` clamped at both ends — rank 57 of 60, where the first draft's `round(fraction × n + 0.5)` read rank 58 and printed a figure one sample away from the one it claimed. `POS_LATENCY_BUDGET` overrides the budget and the sale count is fixed, so two runs are comparable.
  - **repeatable in CI or a scheduled run** — it is a `tests/check_*.py`, so the backend's pipeline loop runs it on every change with no extra wiring; and the trades it made are still whole afterwards (the shift's Z-Report still ties and the day report agrees over the 120 sales).
  **Not in scope, deliberately**: optimising anything. A failure here is a finding for the owner of whatever made the sale slow, and none was found. **Pipeline run locally for this id, after its last edit**: the check itself green, and the backend's loop over `tests/check_*.py` green (**65/65**) with `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: S
- **Owner Role**: QA / Test Engineer
- **Status**: DONE

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
- **Evidence**: **DONE** — the exit verification, in the **backend repository**, as the one check `tests/check_phase3_exit.py`, run 2026-10-07 against a scratch PostgreSQL 16 (`DATABASE_URL=postgresql+psycopg://… python tests/check_phase3_exit.py`, exit 0, printed result *"all assertions green — Phase 3's exit criteria hold"*). It walks the cycle through the services the phase built — it re-implements no step and re-keys no figure, and every document is reached from the one before it:
  - **the criterion itself, order to cash with no manual re-keying** — opportunity `Acme roller blinds` → quotation `Q-EXIT` (10 units at `90.000000`, priced by the engine under the `GOLD` rule) → order `SO-EXIT` (credit-checked) → shipment `SH-EXIT` → invoice `AR-EXIT` → a `252.000000` receipt → dunning at level `FINAL` → gateway payment `PAY-EXIT` for the `756.000000` left. Each document is read back by its number, and the totals are asserted against each other rather than restated: the same figure the quotation charged is on the invoice and the ledger entry, and nothing in the run was typed twice.
  - **credit limits enforced on the commitment, exposure itemised** — a second order against the same customer is refused by name at the live exposure of `756.000000` against a `10000.000000` ceiling, then confirmed once the ceiling is raised; the exposure it was judged against is itemised to its 4 contributing documents, from the one implementation (T-3.AR.06).
  - **a POS sale end to end, shift and Z-Report included, postings balanced** — the till sold 20 sales on one shift and closed it: the Z-Report ties (`4032.000000` gross = `3600.000000` net + `432.000000` tax), the day report over the same shift agrees to the figure, and the shift's close reconciles against its own expected cash rather than a stored snapshot. The 46 journal entries the cycle wrote all balance with two lines or more, and the 23 stock movements are read from the stock ledger.
  - **the latency report** — §6 metric 6 over the gate's own 20 complete sales: min `32 ms`, median `35 ms`, p95 `59 ms`, max `59 ms`, against the `2000 ms` budget. The full distribution (120 sales, two passes) is the one `tests/check_pos_latency.py` prints under T-3.POS.06.
  - **the subledger equals the control account** — after the whole cycle the AR subledger equals the receivables control account to the last decimal (`756.000000`), and the aging report's own total is that same figure, computed from the open items rather than stored.
  **Not in scope, deliberately**: fixing anything the gate finds (a failure here is a finding for the owner of the offending task, and there were none) and the two Phase 6 hardening reviews. **Pipeline run locally, after the batch's last edit**: the backend's loop over `tests/check_*.py` green (**65/65**) with `tests/check_ledger_integrity.py` green, and this check among them.
- **Estimated Effort**: M
- **Owner Role**: QA / Test Engineer
- **Status**: DONE

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
- **Evidence**: **DONE** — the multi-level BOM, its scrap and its explosion, in the **backend repository**. New module `app/manufacturing/bom.py` (`Bom`, `BomLine`, `create_bom`, `add_line`, `release`, `revise`, `explode`, `lines_of`), the one check `tests/check_bom.py` (five sections):
  - **a three-level BOM explodes to the right quantities with scrap at every level** — the check's tree is four levels deep and hand-checkable: 10 bicycles take `10.500000` frames (1 each, 5 % scrapped), `20.000000` wheels (2 each, nothing uplifted), `31.500000` tubes (3 per *scrapped* frame), `22.000000` tyres (10 % of their own), `100.000000` spokes and `63.000000` billets (2 per tube). The frame's 5 % is what the tube's requirement is computed on, so scrap **multiplies through** the levels rather than being added at the end.
  - **a circular BOM is rejected, and a component cannot be its own ancestor** — three refusals with the path named: the bicycle as its own component (*"cannot be a component of its own BOM"*), the frame added to the tube made from it, and the bicycle added two levels up (*"would close a loop: … is already made from …"*). `_refuse_cycle` walks the items reachable *down* from the component; the explosion's own `CircularBomError` is the backstop, not the detection.
  - **scrap of zero is not "unset"** — the wheel's line states `0.0000` and the tube's states nothing; both take no uplift, the explosion reports `scrap_percent: Decimal('0.0000')` against `None`, and the wheel's requirement is `20.000000` (not 21).
  - **a BOM in use cannot be silently changed** — editing the released wheel BOM is refused (`BomLockedError`); `revise` produces the next version as a **draft** carrying the same make-up, the released version still explodes to the same figures after the revision is edited, and the revision's own explosion moves (`22.000000` → `16.000000` tyres per 10 units once a half-tyre line is added).
  - **the walk and the totals agree** — the level list summed per item equals `required` for all six components.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**66/66**) and `tests/check_ledger_integrity.py` green. No contract or frontend change (this id adds no HTTP surface).
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: DONE

#### Task ID: T-4.BOM.02
- **Title**: Routing — operations and their sequence
- **Description**: Attach operations in sequence to a BOM, each with its work center, setup and run time, and the components consumed at that operation.
- **Plan source**: §2.4 Core Components (… and routing), §4 Phase 4 bullet 1, §5 Operation
- **Scope**: Routing definition and sequencing. Capacity and rates are T-4.WC.01.
- **Variables / Config**: `operation_sequence`, `uom_conversion_factor`.
- **Dependencies**: T-4.BOM.01
- **Acceptance Criteria**: Operations carry a unique sequence within the BOM; times are expressed per unit and per batch consistently and the basis is labelled; a component can be assigned to a specific operation or to the BOM generally, and both are reported; a routing cannot have a gap or duplicate sequence.
- **Evidence**: **DONE** — the routing and its sequencing, in the **backend repository**. New module `app/manufacturing/routing.py` (`RoutingOperation`, `add_operation`, `assign_component`, `operations`, `routing`, `operation_minutes`, `routing_minutes`, `unassigned`, `sequence_check`), the one check `tests/check_routing.py` (five sections):
  - **a four-operation routing, validated and displayed in order** — Cut tube (CUT), Weld frame (WELD), Paint (PAINT), Assemble (unassigned), sequences `[1, 2, 3, 4]`; the paint kit is assigned to step 3 and shown under it, while the frame and the wheel are reported as the BOM's own components — **both groups**, so neither hides the other.
  - **a sequence cannot repeat or leave a hole** — a second operation at sequence 2 is refused (`DuplicateOperationError`) and sequence 7 when 5 was expected is refused naming the expected number (`SequenceGapError`); appending gives `[1, 2, 3, 4, 5]`, which `sequence_check` reports as dense with no duplicate.
  - **times state their basis, and both bases are accepted** — the report carries `setup_basis: per_batch` and `run_basis: per_unit`; Weld was timed *12 minutes for 4 units*, is stored as `3.000000` a unit, and `run_basis_minutes` still returns the stated `12.000000`. A batch of 10 is hand-checked: `15 + 2×10`, `20 + 3×10`, `30 + 5×10`, `10 + 8×10` and `0 + 1×10` sum to `265.000000` minutes — setup once each, the run per unit.
  - **a released BOM's routing is frozen** — adding an operation and re-assigning a component are both refused (`RoutingLockedError`), so the route changes by revising the BOM, exactly as its lines do.
  - **an unassigned operation is reported** — Assemble names no work centre and `unassigned` returns it, rather than capacity planning loading it onto nobody.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**67/67**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `WC` — Work Centers & Capacity (§2.4)

#### Task ID: T-4.WC.01
- **Title**: Work centers — capacity, hourly rates, downtime
- **Description**: Model work centers with their available capacity per period, hourly cost rate and downtime allowance.
- **Plan source**: §2.4 Core Components (Work Centers & Operations (capacity, hourly rates, downtime)), §4 Phase 4 bullet 2
- **Scope**: Work center master data. Basic capacity planning is T-4.WC.02; advanced planning is Phase 6.
- **Variables / Config**: `work_center_capacity`, `work_center_hourly_rate`, `downtime_percent`.
- **Dependencies**: T-4.BOM.02
- **Acceptance Criteria**: Capacity is expressed per period and the period is stated; downtime reduces the effective capacity used downstream; hourly rate changes are dated so historical costing is not restated; a capacity or rate of zero is rejected or explicitly flagged as invalid.
- **Evidence**: **DONE** — work centre master data, in the **backend repository**. New module `app/manufacturing/work_centers.py` (`WorkCenter`, `WorkCenterRate`, `create_work_center`, `set_rate`, `rate_on`, `rate_history`, `effective_capacity_minutes`, `capacity_of`, `work_center_by_code`), the one check `tests/check_work_centers.py` (five sections):
  - **two centres with different capacities, each stating its own period** — CUT is `480.000000` minutes **per day**, WELD is `2400.000000` **per week**, and `capacity_of` reports the figure with the unit so neither is read as the other.
  - **downtime reduces the capacity downstream loads** — CUT's 10 % allowance leaves `432.000000` minutes of its 480 (`effective_capacity_minutes`, what T-4.WC.02 loads and judges overload against), while WELD, which loses none, is unchanged.
  - **a dated rate change that does not alter a past job's cost** — CUT rated `250.00` from 2026-01-01 and `310.00` from 2026-07-01; a two-hour job finished in March is worth `500.000000` **both before and after** the July rate landed, work after July prices at `310.000000`, and a second rate for a date already rated is refused (`RateAlreadyDatedError`) so the figure a past job was costed at stays on the record.
  - **a zero is refused, and the refusal names the field** — capacity `0` (*"a centre nobody can load is a wrong figure, not a decision"*), an hourly rate of `0` (*"a rate nobody stated is not a rate of nothing"*), and a period nobody recognises (*"capacity is stated per one of day, week, month"*).
  - **nothing stated is nothing to state** — an unrated centre reads `None` rather than a guessed rate, and the dated history lists both rates still on the record.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**68/68**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-4.WC.02
- **Title**: Basic capacity planning
- **Description**: Load planned and open work orders onto work centers over a horizon and show overload/underload per period.
- **Plan source**: §2.4 Key Capabilities (Capacity planning and costing), §4 Phase 4 bullet 2
- **Scope**: Basic load calculation and view over the horizon. Constraint-based levelling and finite scheduling are Phase 6 (T-6.ADV.01).
- **Variables / Config**: `mrp_horizon_days`, `work_center_capacity`, `downtime_percent`.
- **Dependencies**: T-4.WC.01
- **Acceptance Criteria**: Load equals the sum of operation times of the work orders assigned to each work center, adjusted for downtime; overload is visible per period against stated capacity; an unassigned operation is reported rather than ignored; the calculation is reproducible from the work orders.
- **Evidence**: **DONE** — basic capacity planning, in the **backend repository**. New module `app/manufacturing/capacity.py` (`capacity_profile`, `period_capacity`, `load_for`), the one check `tests/check_capacity_planning.py` (five sections):
  - **one overloaded period and one that is not, reconciled by hand to the source work orders** — CUT (480 minutes a day, 10 % downtime → 432) carries `630.000000` minutes on 2026-09-10 against 432, which is `(15 + 10×20) + (15 + 10×40)` for WO-A and WO-B exactly; 2026-09-11 carries `215.000000` for WO-C and is not overloaded. The report names both orders per cell and says each was dated by its `due_on`.
  - **capacity is per period and prorated to the bucket** — WELD (1400 minutes a **week**, no downtime) contributes `200.000000` minutes to a one-day bucket, with the gross figure and the source period beside it rather than implied.
  - **an unassigned operation and an unknown centre are reported, not dropped** — WO-D's step with no work centre (`15.000000` minutes) and its step naming `GHOST` are both listed with their minutes and their order, and the centres carry exactly `1130.000000` minutes, so nothing was loaded onto a bench that does not do it.
  - **a completed order stops loading** — WO-A walked to `closed` and CUT on the 10th fell from `630.000000` to `415.000000`, no longer overloaded: the chart is work still to do.
  - **reproducible** — the same horizon answered the same figures twice, and one cell is `load_for(..., 'CUT', 2026-09-11) == 215.000000` recomputed from the orders.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**70/70**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

### Stream: `WO` — Shop Floor Control (§2.4)

#### Task ID: T-4.WO.01
- **Title**: Work order creation from BOM and demand
- **Description**: Create work orders for a finished item and quantity, snapshotting the BOM/routing version used and expanding component requirements.
- **Plan source**: §2.4 Core Components (Shop Floor Control (Work Orders …)), §4 Phase 4 bullet 3
- **Scope**: Work order creation and requirement expansion (manual or MRP-suggested). MRP suggestion generation is T-4.MRP.02.
- **Variables / Config**: `scrap_percent`, `uom_conversion_factor`.
- **Dependencies**: T-4.BOM.01, T-4.WC.01
- **Acceptance Criteria**: Requirements equal the BOM explosion for the ordered quantity including scrap at every level; the work order records the exact BOM/routing version so later BOM edits do not change it; a work order without a routing can still be created only if the plan allows (not stated) — otherwise the absence is reported rather than assumed; status transitions are auditable.
- **Evidence**: **DONE** — work order creation and requirement expansion, in the **backend repository**. New module `app/manufacturing/work_orders.py` (`WorkOrder`, `WorkOrderRequirement`, `WorkOrderOperation`, `create_work_order`, `advance`, `reconcile_requirements`, `missing_route`, `bom_of`), the one check `tests/check_work_orders.py` (five sections):
  - **the requirements are the BOM explosion, including scrap at every level** — WO-1 for 10 bicycles carries `10.500000` frames (1 each, 5 % scrapped), `20.000000` wheels, `31.500000` tubes and `63.000000` billets, each with its own level (1, 1, 2, 3) and path (`BICYCLE → FRAME → TUBE → ALLOY`), and `reconcile_requirements` reports `matched: True` because both sides are the same `explode` call over the pinned version.
  - **the exact version is pinned, and a later BOM edit leaves it unchanged** — the item's BOM was revised to v2 and its wheel line changed from 2 to 3; WO-1 still names v1, still requires `20.000000` wheels, still reconciles, and still routes the two operations it was raised with (the route is copied onto the job, so a revision cannot reach it either).
  - **the absence of a route is reported rather than assumed** — WO-2 was raised against a released BOM with no routing: it exists, `missing_route` answers `True` and its operation list is empty. An item with **no released BOM** is refused (`NoBomError`: *"nothing to expand"*), because inventing requirements would be worse than refusing.
  - **status transitions are a transition, not an assignment** — the check walks `planned → released → in_progress`, refuses the move back to `planned` with the statuses that *do* follow, and refuses a status nobody defines.
  - **every move is auditable** — the trail holds both moves for the row with the before and the after (`('planned', 'released')`, `('released', 'in_progress')`), written by the table's own trigger (T-0.AUDIT.02) rather than by this module.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**69/69**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
  - **Corrected by the Phase 4 gate (T-4.X.GATE, same run)**: `app/manufacturing/work_orders.py` gained `direct_requirements` (the job's **own**, level-1 components) beside `requirements_of` (the whole explosion the order was raised from, which the criterion above is about). Building the gate's chain exposed that the requirement list alone cannot answer *what this job consumes*: with the explosion's rows treated as the draw, a bicycle job demanded the tubes its frames are made of *as well as* the frames, so the same 55 tubes would be consumed twice (once by the frame job, once by the bicycle job). The list is unchanged — `requirements_of` and `reconcile_requirements` are untouched, and so is this check's expectation of levels 1–3 — and the draw is now stated separately. **Pipeline run locally after that correction**: the backend's loop over `tests/check_*.py` green (**78/78**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-4.WO.02
- **Title**: Job cards, time tracking and production output
- **Description**: Issue job cards per operation, record time booked (setup and run) and produced quantities including rejects, per operator and work center.
- **Plan source**: §2.4 Key Capabilities (Time tracking and production output), §2.4 Core Components (… Job Cards …), §4 Phase 4 bullet 3
- **Scope**: Job card execution and time/output capture. Costing of the captured time is T-4.WO.05.
- **Variables / Config**: `operation_sequence`, `work_center_hourly_rate`.
- **Dependencies**: T-4.WO.01
- **Acceptance Criteria**: Time is booked per operation and cannot exceed a configured limit without acknowledgement; produced and rejected quantities are recorded separately and reconcile to the work order quantity; total booked time equals the sum of job card entries; a job card cannot be edited after the operation is closed except by an audited correction.
- **Evidence**: **DONE** — job card execution and time/output capture, in the **backend repository**. New module `app/manufacturing/job_cards.py` (`JobCard`, `JobCardEntry`, `open_card`, `book_time`, `correct_entry`, `close_card`, `booked_time`, `output_of`, `time_by_work_center`), the one check `tests/check_job_cards.py` (five sections):
  - **total booked time is the sum of the entries**, per card and per order — three cards and five bookings: ana's card reads `15.000000` setup + `60.000000` run, ben's `45.000000`, and the order's `179.000000` minutes equal the sum over its cards.
  - **produced and rejected are recorded apart and reconcile to the order** — `11.000000` produced, `1.000000` rejected, a net of `10.000000` against the `10` ordered, with the `0.000000` difference stated while the job is still running.
  - **an overrun past the limit is refused unless somebody owns it** — the limit is the **operation's** planned minutes (`115.000000`) plus 10 %, so two cards on one step share one budget rather than each getting a fresh one; ten more minutes are refused (`OverrunNotAcknowledged`) and accepted once ben acknowledges, with `overrun_acknowledged_by` and `overrun_reason` kept on the entry.
  - **a closed card takes no more time, and a correction is a new entry** — booking is refused after the close (`CardClosedError`), and `correct_entry` appends a row that names the entry it corrects with who and why, leaving the corrected booking's own minutes unchanged beside it.
  - **the time is reported per work centre** — `{'CUT': 130.000000, 'WELD': 49.000000}`, the figure T-4.WO.05 prices at each centre's dated rate.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**71/71**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-4.WO.03
- **Title**: Material issue to work orders
- **Description**: Issue components from stock to a work order, recording the location, quantity, UOM and value, and posting the WIP movement.
- **Plan source**: §2.4 Core Components (… Material Issues …), §4 Phase 4 bullet 3
- **Scope**: Issue and the WIP posting. Substitution/alternate materials are not named in the plan.
- **Variables / Config**: `uom_conversion_factor`, `costing_method` (issued value), `warehouse_hierarchy_levels`.
- **Dependencies**: T-4.WO.01, T-1.INV.05
- **Acceptance Criteria**: Issued quantities are tracked against requirements with the remaining requirement visible; over-issue beyond requirement plus tolerance is refused or explicitly overridden and recorded; the issue creates stock ledger entries and a balanced WIP posting; issued value uses the item's configured costing method.
- **Evidence**: **DONE** — issuing material to work orders and the WIP it books, in the **backend repository**. New module `app/manufacturing/issues.py` (`WorkOrderIssue`, `issue_material`, `outstanding`, `issued_quantity`, `issued_value`, `issue_lines`, `issued_to_wip`) plus one line in `app/stock/gl_posting.py` (`work_order_issue` → the `work_in_progress` key), the one check `tests/check_material_issue.py` (five sections):
  - **issued quantities are tracked against the requirement** — after issuing `6.000000` of the `10` required, `outstanding` reads required `10.000000`, issued `6.000000`, remaining `4.000000`, and the figure is the sum of the issue rows rather than a balance kept beside them.
  - **the issue writes a stock ledger row and a balanced WIP posting** — one movement (`-6.000000` at `-60.000000`, valued by the costing method at ten a unit) and one entry touching `{'1200': -60.000000, '1230': +60.000000}` with debits equal to credits; the bin falls from 30 to `24.000000`.
  - **an over-issue beyond the requirement plus tolerance is refused, then overridden and recorded** — a fourth issue would have taken the job to `11.25` against a `10.5` ceiling (5 %), refused with `OverIssueError`, and accepted once maria owned it: `overridden`, `override_actor` and `override_reason` are all on the issue row.
  - **what the job did not call for, and what the store does not hold** — issuing the produced item itself is refused (`NotRequiredError`: *"does not call for"*), and so are thirty blanks from a bin holding `18.750000` (`InsufficientStockError`), so the stock side is T-1.INV.05's real one.
  - **the WIP the order carries is the sum of its issues** — `112.500000` over three issues (`60.000000`, `42.500000`, `10.000000`), the figure T-4.WO.04 clears.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**72/72**) and `tests/check_ledger_integrity.py` green. A first attempt ran concurrently with the next id's run against the same scratch database and deadlocked on it; the run was repeated sequentially (at that point `73/73`, including the next id's check) and is green.
  - **Corrected by the Phase 4 gate (T-4.X.GATE, same run)**: `outstanding` (and therefore `consumption_report`) still lists **every** requirement with what was issued against this order, and its docstring now says why a row below level 1 usually shows nothing issued: that material is drawn by the component's own work order (`direct_requirements`). No figure in this check moved — its dataset is one level deep. **Pipeline run locally after that correction**: the backend's loop over `tests/check_*.py` green (**78/78**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-4.WO.04
- **Title**: Finished goods receipt from a work order
- **Description**: Receive the produced item into stock from the work order, consuming the corresponding component quantities and clearing WIP.
- **Plan source**: §2.4 Core Components (… Finished Goods), §4 Phase 4 bullet 3
- **Scope**: Production receipt and its stock effect. Cost calculation is T-4.WO.05.
- **Variables / Config**: `uom_conversion_factor`, `costing_method`, `warehouse_hierarchy_levels`.
- **Dependencies**: T-4.WO.03
- **Acceptance Criteria**: Receiving the full quantity consumes exactly the BOM requirement (within tolerance) and clears the work order's WIP to zero; partial receipts consume proportionally and leave the balance consistent; a receipt that would leave unreconciled WIP is reported rather than silently accepted; every movement appears in the stock ledger.
- **Evidence**: **DONE** — production receipts, the material they consume and the WIP they clear, in the **backend repository**. New module `app/manufacturing/receipts.py` (`WorkOrderReceipt`, `receive_finished_goods`, `consumption_report`, `receipt_lines`, `wip_balance`) plus one line in `app/stock/gl_posting.py` (`work_order_receipt` → `work_in_progress`), the one check `tests/check_finished_goods.py` (five sections):
  - **a partial receipt consumes its share and takes its share out of WIP** — receiving 4 of 10 widgets drew `8.000000` of the `20.000000` blanks the job takes and `32.000000` of the `80.000000` issued for it, leaving a WIP balance of `48.000000` derived from the issues and the receipts.
  - **the final receipt consumes exactly the BOM requirement and clears WIP to zero** — the last 6 units consumed the remaining `20.000000` blanks and took `168.000000`; WIP ends at `0.000000` with `10` widgets in stock, so nothing is left unreconciled.
  - **the stock ledger and the ledger both show it, to the last decimal** — the movements name the receipts as their documents, inventory stands at `400.000000` (the blanks' 400 back as ten widgets) and the WIP account at `0.000000`, with both receipt entries balanced.
  - **a receipt nobody has drawn for is refused, naming the shortfall** (`ConsumptionNotCovered`: *"BLANK 4.000000"*), as is one that would report more than the order was raised for (`OverReceiptError`), and one before the job is being worked (`WorkOrderNotRunning`).
  - **the consumption report** a cost accountant reads before closing: received `10.000000` of `10.000000`, WIP `0.000000`, `within_tolerance: True`, `complete: True`, every requirement issued exactly.
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**73/73**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
  - **Corrected by the Phase 4 gate (T-4.X.GATE, same run)**: the receipt's consumption — both the backflush (`_consume_proportionally`) and the coverage check a receipt without one makes (`_require_covered`) — is judged on the job's **direct** components (`direct_requirements`, T-4.WO.01) rather than on every row of the explosion: a job consumes what its own item is made of, and the materials of a component that is itself built are that component's own job's to draw. Without this the gate's bicycle job could not have been received without consuming the frame job's tubes a second time. This check's dataset is one level deep, so its five sections are unchanged and still green. **Pipeline run locally after that correction**: the backend's loop over `tests/check_*.py` green (**78/78**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: M
- **Owner Role**: Backend Engineer
- **Status**: DONE

#### Task ID: T-4.WO.05
- **Title**: Production costing and its GL posting
- **Description**: Cost a work order from issued material value plus operation time at work center rates, compare to the standard/expected cost, and post the variance.
- **Plan source**: §2.4 Key Capabilities (Capacity planning and costing), §4 Phase 4 exit criteria (correct costing)
- **Scope**: Cost roll-up, variance and posting. No overhead allocation models beyond the named hourly rates (not named in the plan).
- **Variables / Config**: `work_center_hourly_rate`, `costing_method`, `scrap_percent`.
- **Dependencies**: T-4.WO.04, T-4.WC.01, T-1.ACCT.03
- **Acceptance Criteria**: Work order cost equals material issued value plus booked time at the recorded (dated) rates; scrap quantity is attributed to cost, not lost; the variance against expected is computed and posted, and the posting balances; recosting the same work order twice produces no duplicate posting; the finished goods value equals the work order cost.
- **Evidence**: **DONE** — production costing and its GL posting, in the **backend repository**. New module `app/manufacturing/costing.py` (`WorkOrderCost`, `cost_work_order`, `cost_breakdown`, `labour_breakdown`, `cost_summary`, `finished_goods_value`) plus the `labour_applied` / `production_variance` mapping keys, the one check `tests/check_production_costing.py` (five sections):
  - **the cost is material plus booked time at the recorded (dated) rates** — material `200.000000` (20 blanks at ten a unit, scrap already inside the requirement) plus labour `807.500000`: `120.000000` minutes on CUT at the `250.000000` in force = `500.000000`, `45.000000` on WELD at `410.000000` = `307.500000`; the job's cost is `1007.500000` against a standard of `500.000000`.
  - **the finished goods carry the job's own cost** — the receipts had capitalised the `200.000000` of material they took out of WIP, and the costing added the `807.500000` of labour they could not yet know, so the goods stand at `1007.500000` rather than a material-only figure.
  - **the variance is computed and posted, and the posting balances** — one entry: inventory debited `807.500000`, `labour_applied` credited `300.000000` (the standard less the material the receipts carried — what the standard allowed the output), `production_variance` credited `507.500000` (the difference); debits equal credits, and inventory reads `1407.500000` by hand (600 of blanks, 200 issued, 200 of goods back, 807.50 of labour).
  - **costing twice posts once** — a second call returns the recorded row (`1007.500000`) and the ledger still holds one entry for the order; `counted` reports one costing.
  - **refusals** — a job whose output has not all been received is refused (`WorkOrderNotComplete`), and an item with **no standard cost** to be judged against is refused (`MissingStandardCostError`).
  **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**74/74**) and `tests/check_ledger_integrity.py` green. No contract or frontend change.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: DONE

### Stream: `MRP` — Material Requirements Planning (§2.4)

#### Task ID: T-4.MRP.01
- **Title**: MRP engine — net requirement calculation (sales orders vs stock)
- **Description**: Compute net requirements by exploding demand (sales orders) against on-hand and open supply, applying BOM levels, scrap and lead time, over the configured horizon.
- **Plan source**: §2.4 Core Components (Material Requirements Planning (MRP) engine), §2.4 Key Capabilities (Net requirement calculation (Sales Orders vs Stock)), §4 Phase 4 bullet 4
- **Scope**: The netting calculation and its output. Converting output to suggested orders is T-4.MRP.02; accuracy verification is T-4.MRP.03.
- **Variables / Config**: `mrp_horizon_days`, `mrp_bucket`, `mrp_demand_sources` (Sales Orders vs Stock), `scrap_percent`.
- **Dependencies**: T-3.SALES.04, T-1.INV.04, T-4.BOM.01
- **Acceptance Criteria**: Net requirement equals demand minus available stock and open supply at every BOM level, with scrap applied and lead times respected across buckets; a shortage at a sub-level propagates to the parent requirement; running MRP twice on unchanged data produces identical results (deterministic); the calculation states its inputs (horizon, bucket, demand sources) on every run.
- **Evidence**:
  - Implemented in `app/manufacturing/mrp.py` (`run_mrp`, `plan_of`, `plan_sorted`, `inputs_of`, `runs_of`; tables `mrp_run`, `mrp_requirement`), exercised by `tests/check_mrp.py` — a two-level dataset (WIDGET made from 2 BLANK each, BLANK bought with a 3-day lead time; 5 blanks on the shelf; SO-1 for 10 widgets; WO-1 open for 4 widgets, wanted in the first bucket).
  - **net requirement, hand-checked bucket by bucket** — WIDGET bucket 1: gross `10.000000`, supply `4.000000` (the open job), net `6.000000`; BLANK bucket 1: gross `12.000000` (6 × 2), available `5.000000`, net `7.000000`. Both rows carry their level (`0`, `1`) and kind (`make`, `buy`); the plan holds exactly those two rows.
  - **a sub-level shortage reaches the parent** — the WIDGET row is `constrained` with `constrained_by="BLANK"`: the parent that cannot be built says so on its own line rather than leaving it to be inferred from the level below.
  - **lead times respected** — the BLANK is wanted in the week of `2026-10-05` and its requirement states `release_on = 2026-10-02`, i.e. the 3-day lead time applied to the bucket (`lead_time_days = 3`); the made item's release date is its own bucket.
  - **the run states its inputs** — `inputs_of(run)` = `{start: 2026-10-05, horizon_days: 28, bucket_days: 7, demand_sources: ('sales_orders',), run_on: 2026-10-05}`, and the second feed (`work_orders`) really is demand: the open job owes `8.000000` blanks, `5.000000` on the shelf, net `3.000000`, with no WIDGET row; a feed this system does not have (`forecasts`) is refused (`UnknownDemandSource`).
  - **deterministic** — a second run over unchanged data produced the identical plan (`plan_sorted` equal row for row, 2 rows), as a distinct `MrpRun`.
  - **Pipeline run locally for this id, after its last edit**: the backend's loop over `tests/check_*.py` green (**75/75**) and `tests/check_ledger_integrity.py` green. No contract or frontend change (no HTTP surface added).
  - **Corrected by the Phase 4 gate (T-4.X.GATE, same run)**: the `work_orders` demand feed (`_work_order_demand`) now reads each open job's **direct** components (`direct_requirements`, T-4.WO.01) instead of every row of its explosion: a released job draws what its own item is made of, and counting the deeper rows as well would demand the same material twice — the frame job's 55 tubes *and* the bicycle job's 55 again. The feed's own section-4 figures are one level deep and unchanged in this check. **Pipeline run locally after that correction**: the backend's loop over `tests/check_*.py` green (**78/78**) and `tests/check_ledger_integrity.py` green.
- **Estimated Effort**: L
- **Owner Role**: Backend Engineer (with Domain Analyst — Manufacturing)
- **Status**: DONE

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

### Resolved 2026-09-18

1. **`container_registry`** — **`ghcr.io/paws1234`, public packages** (decided by the user for `T-0.CICD.01`, and the value both pipelines publish to: `ghcr.io/paws1234/erpv1-backend:<commit>` and `ghcr.io/paws1234/erpv1-frontend:<commit>`, both pullable anonymously). The Variables table states it instead of "Not stated". *(Affects `T-0.CICD.01`, `T-0.CICD.02`, `T-0.DEPLOY.03`.)*
2. **Ledger drift, corrected in `T-0.X.GATE`** — the `## Repositories` table said all three repositories had **no commits yet** and the paragraph called the remotes empty (they have commits); the routing table still called the deployment stack's home undecided although `T-0.DEPLOY.03` had already landed it in the backend repository. All three are now stated as they are.

### Still open

1. **Philippines fiscal year start** — needed by the CoA, period locking, the statements and payroll; it must be confirmed with the pack in `T-0.LOC.01` rather than assumed. *(Affects `T-0.LOC.01`, `T-1.ACCT.01`, `T-1.ACCT.04`, `T-1.ACCT.07`.)*
2. **Job-queue library** — Postgres-backed is decided; `pgqueuer` vs `procrastinate` is not pinned. *(Affects `T-0.REPORT.01`, `T-0.CICD.01`, `T-3.AR.03`, `T-3.AR.04`, `T-6.OFFLINE.01`, `T-6.OFFLINE.02`.)*
3. **Remaining configuration defaults** — deferred by decision 7b to the phase that needs each: `fx_rate_source`, `approval_thresholds`, `three_way_match_tolerance`, aging buckets, `dunning_levels`, `credit_check_mode`, `mrp_horizon_days` / `mrp_bucket`, `payroll_cutoff_day`, `shift_definitions`, `overtime_rules`, `leave_accrual_rule`, supplier scoring weights, `report_schedule`.
4. **Deployment target for the container stack** — the plan fixes one container per service and a single compose file, but not *where* they run (a single host, or a managed database plus app hosts). Compose covers local development and a single-host deploy; anything beyond that needs a decision before `T-0.DEPLOY.03` is used in anger. *(Affects `T-0.DEPLOY.03`, `T-0.CICD.01`, `T-6.HARD.01`.)*
5. **Home of the compose/deployment stack** — narrowed 2026-09-17: only two repositories were created, so the stack file lands in one of them rather than in a third infrastructure repository — most naturally the backend repository, which owns the service definitions. **Confirm which.** *(Affects `T-0.DEPLOY.03`, `T-0.CICD.01`, `T-0.CICD.02`.)* — **Update 2026-09-18**: `T-0.DEPLOY.03` is `DONE` and its file landed in the **backend** repository, in the repository the user named for it; what remains open is only confirming that this is where it should stay.

*(Two items previously listed here are settled as of 2026-09-17: the plan and ledger live in the planning repo `paws1234/ERPPLANNING` (`ERPV1/.github`), and the app repos are `paws1234/ERPbackend` and `paws1234/ERPfrontend` — see the Repositories section.)*

---

*Every task above is `TODO`. Statuses are owned by `/do-task` and must not be changed here.*
