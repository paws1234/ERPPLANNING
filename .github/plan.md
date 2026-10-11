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

**Production Principles** (added 2026-10-10, with Phases 7–15; binding on every task from here)

The domain and the ledger reached production grade in Phases 0–6. They are wrapped in a prototype-grade seam — no identity, no migrations, no worker, no logs, no backups. These principles close that seam, and each one is a rule a task can be failed against:

1. **One identity, stated once.** No route reads `X-Actor` or `X-Company-Id` after Phase 7. The authenticated session is the only source of company and actor; an unauthenticated request answers `401`, never `unknown`.
2. **The schema evolves by migration, never by `create_all`.** Alembic owns the schema from Phase 8; a new table that arrives without a migration fails the build, and a migrated schema is company-scoped exactly like a `create_all` one.
3. **Every unit of background work is scheduled, idempotent, observable and retryable.** A job that ran twice is safe, and the second run is visible as such.
4. **Nothing fails silently.** Every failure is logged with a correlation id, answers in the one documented error shape without internals, and increments a counter an operator can see.
5. **Operational truth is tested, not asserted.** Readiness, restore, migration upgrade, rate limits and job firing are each proven by a check that would go red if the behaviour broke. Documentation is not evidence.
6. **Configuration is validated at startup and read once.** A misconfigured instance refuses to start rather than starting "healthy" and failing per request.
7. **Secrets come from the environment, never from a default**, and are rotatable without a rebuild.
8. **API-first stands.** A new route is published to the contract artifact before the frontend consumes it.

**Technology Stack** (decided 2026-09-17)
- Backend: Python — FastAPI + SQLAlchemy
- Frontend: Next.js
- Database: PostgreSQL
- Job queue: Postgres-backed, no Redis (library pinned when the first scheduled job is built)
- Search: PostgreSQL full-text

**Repositories** (routing — every module and phase below lands in exactly one)

| Repository | Local path | Remote | What lands in it |
|---|---|---|---|
| Backend | `ERPV1/ERPbackend` | `paws1234/ERPbackend` | Every server-side module, the Alembic tree, the worker, `Caddyfile`, the compose stack, backups, PDF and statutory output, lint/type configuration, `pyproject.toml` |
| Frontend | `ERPV1/ERPfrontend` | `paws1234/ERPfrontend` | The login and session shell, admin screens, audit viewer, i18n, error/loading/not-found boundaries, ESLint/Prettier, E2E and accessibility tests |
| Planning (this plan's home) | `ERPV1` | `paws1234/ERPPLANNING` | `.github/plan.md` (this plan) and `.github/tasks.md` (the ledger); both application folders are gitignored and never committed here |

Server-side code never lands in the frontend repository and vice versa. The compose/deployment stack stays in the **backend** repository, where `T-0.DEPLOY.03` put it (the ledger's open item 5, disposed 2026-10-10).

**High-Level Modules**

Domain modules (Phases 1–6, complete):

1. Accounting & Financial Management
2. Inventory & Warehouse Management (WMS)
3. Supply Chain & Procurement
4. Manufacturing & Production (MRP / MES)
5. Sales, CRM & Point of Sale (POS)
6. Human Resources & Payroll (HRMS)

Operational modules (added 2026-10-10 — Phases 7–15):

7. Identity & Access
8. Administration & Tenancy
9. Schema Evolution
10. Background Work (worker & scheduler)
11. Observability & Operations
12. Delivery & Messaging
13. Edge, Secrets & Backup
14. Document Control & Fiscal Close
15. Data Integrity & API Hardening
16. Engineering Standards & Assurance
17. Frontend Operational Shell
18. Onboarding & Legacy Migration
19. Functional Completeness (gated — Phase 15)

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
| Identity                  | Server-side sessions (Argon2id credentials, TOTP MFA); the session is the only source of actor and company — added 2026-10-10 |
| Schema Evolution          | Alembic migrations; `create_all` only for tests and the baseline; RLS applied by the migration — added 2026-10-10 |
| Background Work           | `pgqueuer` (Postgres-backed); a worker container; every job idempotent with a recorded run — added 2026-10-10 |
| Observability             | Structured JSON logs with correlation ids; liveness/readiness split; `/metrics`; no internals in a `500` — added 2026-10-10 |
| Delivery & Messaging      | Transports registered from configuration at startup; every attempt recorded and retried — added 2026-10-10 |
| Document Control          | Server-side gapless numbering per company, document type and fiscal year; PDF renditions; attachments — added 2026-10-10 |
| Backup & Recovery         | Off-host `pg_dump`, retention, and a rehearsed restore drill against the ledger gate — added 2026-10-10 |

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

## 4b. Phases 7–15 — Production Readiness (added 2026-10-10)

**Source:** an audit of the running system on 2026-10-10 (43 findings: 10 blockers, 17 high, 16 medium). Phases 0–6 built a correct ledger inside a prototype-grade seam; these phases close that seam. Every finding in the audit was spot-verified against the tree before planning (0 `import logging` in `app/`; 29 in-code ceilings; the catch-all `500` echoing `type(exc).__name__`; `DATABASE_URL` read at first request; a three-service compose with one volume; a pipeline that runs checks only).

**Decided with this plan (2026-10-10):** the queue library is **`pgqueuer`**; the deployment target is a **single Linux host running Docker Compose behind Caddy with automatic TLS and off-host `pg_dump` backups**; the compose stack stays in the **backend** repository; authentication is **built in** (email + password, Argon2id, server-side sessions, TOTP MFA).

### Phase 7 – Identity & Access

- **Goal:** the company and actor of every request come from an authenticated session, and identity is administrable through the API.
- **Deliverables** (backend): `app/identity.py` (`User`, `Credential`, `Session`, `MfaSecret`, recovery codes), `app/auth.py` (login, logout, session dependency, company resolution), `app/admin.py` (users, roles, capability grants, field restrictions, company create/suspend), `app/api.py` `context()` rewritten, `app/security.py` taking its subject from the session, the contract republished. (frontend): `app/login/page.tsx`, `lib/session.ts`.
- **Dependencies:** none.
- **Exit Criteria:**
  - Every route except health, login, the contract and the docs answers `401` without a valid session; a check enumerates the route table and asserts it.
  - No route reads `X-Actor` or `X-Company-Id`, and neither appears in the published contract.
  - A session bound to company A receives no row and no `200` for company B on any route.
  - An administrator creates a user and grants a role through the API; the user's next request is permitted, with no deploy and no SQL.
  - A company is created and suspended through the API; a suspended company's users are refused.
  - MFA is enforced for the administrator role; a reused code is refused. Lockout after N failures in the window, and the refusal is in the audit trail.
  - The frontend signs in, survives a reload, and signs out.

### Phase 8 – Platform Operations

- **Goal:** the platform starts safely, evolves its schema, runs its own scheduled work, and can be observed.
- **Deliverables:** `alembic.ini` and `migrations/` (a baseline plus the RLS half of `app/db.py` invoked from the migration environment), `app/queue.py`, `app/worker.py`, `app/schedule.py` on `pgqueuer` with a worker service in `docker-compose.yml`, `app/observability.py` (structured logs, correlation ids), `app/api.py` (health split into liveness and readiness, `500` error hygiene, `/metrics`, the request log), `app/config.py` (every setting declared and validated at startup), transports registered from configuration with a `delivery` row per attempt, `OPERATIONS.md`, and the checks for each.
- **Dependencies:** none (parallel with Phase 7).
- **Exit Criteria:**
  - `alembic upgrade head` on an empty database yields a schema whose RLS policies match the `create_all` schema table by table; `alembic check` reports no drift.
  - A second `upgrade head` is a no-op, and the install CLI runs on top of migrations, idempotently.
  - The worker container is in the stack, and every schedule in `app/schedule.py` has been observed to fire in a check.
  - A job run twice produces one effect; a failing job retries and records the attempt and its error.
  - Readiness answers `503` within the timeout when the database is stopped and `200` when it returns; liveness stays `200` throughout.
  - A forced unhandled exception answers `500` with `internal_error` and a correlation id, and no exception class, message, path or SQL appears in the body.
  - Every request produces one structured log line carrying the correlation id, and a credential never appears in it.
  - With SMTP configured a delivery reaches the sink; without it, the attempt is recorded — never a `500`.
  - A missing or malformed `DATABASE_URL` fails at startup with a named variable.

### Phase 9 – Edge, Secrets & Backup

- **Goal:** the running stack is reachable only over TLS, throttled, backed up off-host and reversible.
- **Deliverables:** `Caddyfile` and a `proxy` service, `app/crypto.py` (envelope encryption, payroll bank details first), `scripts/backup.sh` and `scripts/restore-drill.sh` with retention, compose resource limits and the scaling variables, staging and production deploy jobs with a rollback, release-time contract publication, and the checks for each.
- **Dependencies:** Phase 8.
- **Exit Criteria:**
  - Plain HTTP on the public port redirects to HTTPS with a valid certificate; the backend's port is not published.
  - Requests above the configured rate answer `429` with `Retry-After`; bodies above the limit answer `413`.
  - Payroll bank account numbers are unreadable in a raw table read and readable through the API only to a permitted caller.
  - A backup lands off the host; the restore drill restores it, runs the ledger gate, and finishes within the recorded RTO.
  - A deploy rolls back to the previous image tag, exercised in the pipeline.
  - The release job republishes the contract, and the frontend's pinned artifact matches the released backend.

### Phase 10 – Document Control & Fiscal Close

- **Goal:** documents are numbered, printable, attachable, auditable and closable.
- **Deliverables:** `app/numbering.py` (gapless sequences per company, document type and fiscal year), `app/documents.py` (PDF for invoice, payslip, statement, purchase order, delivery note, receipt), `app/attachments.py`, audit read/query routes, `app/ledger/closing.py` (closing entries, retained-earnings roll-forward, opening balances, reversal), the Philippines statutory artefacts, and the checks for each.
- **Dependencies:** Phase 7 (attachments and audit reads are permissioned), Phase 8 (transports and scheduling).
- **Exit Criteria:**
  - A statutory document's number is allocated by the server; a body-supplied number for that type is refused; 10 000 randomized allocations produce zero gaps and zero duplicates.
  - Numbers are unique per company, document type and fiscal year, and a new fiscal year starts at its own first number.
  - Each document renders to a PDF whose totals reconcile to the JSON that produced it.
  - An attachment uploads, lists against its document, downloads byte-identical, and is refused when its type or size is not allowed.
  - The audit trail is readable through the API with filters and pagination; a caller without the capability is refused.
  - A fiscal year closes: the trial balance still balances, P&L accounts are zero, retained earnings equals the closed result, the balance sheet is unchanged by the close, and the next year opens with the closing balances — asserted from stored rows.
  - A close can be reversed, and the reversal is audited.

### Phase 11 – Data Integrity & API Hardening

- **Goal:** concurrency, retries, list growth, statement reconciliation, rates and devices stop being the weak seams.
- **Deliverables:** `row_version` columns with `409` on a stale write, `Idempotency-Key` on every mutating route, `limit`/`offset` with a maximum page size on every list route, the bank reconciliation workflow (statement import, matching, exceptions), an FX provider adapter with recorded provenance, POS device adapters (drawer, scale, card terminal, label printing) behind interfaces, a disposition for each of the 29 ceilings, and the checks.
- **Dependencies:** Phase 7, Phase 8.
- **Exit Criteria:**
  - Two concurrent writers to one document: the second gets `409` with the current version, and no update is lost.
  - A replayed mutation with the same key and body answers the first result, on every mutating route.
  - Every list route is paginated with a maximum page size (21 of 21).
  - A bank statement imports, matches and reports its exceptions; a reconciliation closes and is reversible.
  - An FX sync records each rate with its source; a provider failure leaves the previous rate in force and is visible.
  - Every recorded ceiling carries a disposition, and each "schedule" entry names the phase that owns it.

### Phase 12 – Engineering Standards & Assurance

- **Goal:** tools check the codebase, and the financial behaviour is protected by tests that would catch a wrong number.
- **Deliverables:** `pyproject.toml` (ruff, mypy, pytest), hashed lockfiles, a pytest runner with a coverage floor, property and golden financial datasets, frontend ESLint/Prettier, Playwright E2E and an axe accessibility pass, Dependabot, `pip-audit`/`npm audit`, CodeQL, CODEOWNERS, tenant export and erasure tooling, and an updated `TESTING.md`.
- **Dependencies:** Phase 7 (the session flow the E2E drives), Phase 8 (the checks it runs).
- **Exit Criteria:**
  - `ruff check`, `ruff format --check` and `mypy` run in CI and pass; a deliberate violation fails the run.
  - Every existing `tests/check_*.py` still runs, now under pytest, with coverage measured and the floor met.
  - The property suite exercises the ledger invariants over randomized postings with no violation; the golden payroll and close sets are stable.
  - A high-severity dependency finding arrives as a Dependabot pull request; `pip-audit` and `npm audit` report nothing outstanding; CodeQL runs on both repositories.
  - Playwright drives sign-in → MFA → admin → sign-out in a browser, and axe reports no critical violation on any page.
  - A tenant export produces a complete artefact; an erasure request anonymises a person without altering a posting, and the ledger gate still passes.

### Phase 13 – Frontend Operational Shell

- **Goal:** the non-domain half of the UI exists, and no page can take the shell down.
- **Deliverables:** the login/MFA/session UI, the company switcher, admin screens (users, roles, grants, companies), the audit trail viewer, the attachments UI, `error.tsx`/`loading.tsx`/`not-found.tsx` per route segment, `lib/i18n.ts` with locale-aware number and date formatting, and a matching `lang` attribute.
- **Dependencies:** Phase 7, Phase 8, Phase 10 (the APIs it consumes), Phase 12 (the harness it is tested with).
- **Exit Criteria:**
  - Sign in, MFA, expiry, sign out and company switch all work against the real API.
  - An administrator creates a user and grants a role from the UI.
  - Every route segment has the three boundaries; a thrown error renders the boundary, not a blank page.
  - Amounts and dates render per locale, and no admin string is hard-coded English.

### Phase 14 – Onboarding & Legacy Migration

- **Goal:** an existing business can be brought on, and a tenant can be provisioned without the database owner's hands on it.
- **Deliverables:** the company provisioning API/CLI on top of migrations, `tools/import_legacy.py` (chart-of-accounts extensions, customers, suppliers, items, BOMs, warehouses, employees, opening balances as one balanced opening entry), `MIGRATION.md` (freeze, export, transform, import, reconcile, go-live, rollback), and its check.
- **Dependencies:** Phase 7, Phase 8, Phase 10.
- **Exit Criteria:**
  - A company is provisioned through the API with its market, fiscal calendar and reporting currency, and the act is audited.
  - An import of masters and opening balances yields an opening trial balance that ties to the source within zero tolerance, and the ledger gate passes afterwards.
  - The cutover runbook has been executed end to end on a faithful export, including its rollback step.

### Phase 15 – Functional Completeness: decision gate

- **Goal:** decide, per candidate module, in or out — and record it.
- **Candidates:** fixed assets & depreciation, budgeting/forecast-vs-actual, treasury/cash management, project & job costing, preventive maintenance, quality management, multi-entity consolidation, e-invoicing/BIR transmission, e-signature.
- **Dependencies:** Phases 7–14 (deliberately last, so new modules are built on a foundation that runs).
- **Exit Criteria:**
  - Each candidate is recorded as IN (with its own phase sketched) or OUT (with a reason), by the user.
  - The two candidates that contradict §8's recorded non-goals — consolidation and quality management — are resolved explicitly.
  - Nothing is built from this gate without a written decision.

**Cut lines.** Phases 7 and 8 alone clear eight of the ten blockers (only TLS and backups remain); Phase 9 clears the rest. The phases are ordered so the plan can be cut at any boundary without leaving a half-built thing behind.

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

Added with Phases 7–15 (2026-10-10):

- **User** — email (unique per install, lower-cased), display name, status, company memberships, MFA state; owns credentials and sessions
- **Credential** — Argon2id hash, its parameters, failure count, locked-until; never a plaintext password
- **Session** — user, active company, issued/expires/revoked, user agent and address; a row, so signing out is real
- **MfaSecret / RecoveryCode** — the TOTP secret (encrypted) and hashed recovery codes
- **Company status** — active or suspended; creatable through the API (§2.1's Phase 7 work)
- **JobRun / Schedule** — job, scheduled-for, started, finished, status, attempt, error, correlation id
- **Delivery** — channel, destination, payload reference, attempt, status, next attempt
- **DocumentSequence** — company, document type, fiscal year, next number; the gapless guarantee lives here
- **Attachment** — document reference, filename, content type, size, checksum, storage key, uploaded-by
- **FiscalYearClose** — company, fiscal year, closed-at, closed-by, the entries it produced, its reversal
- **row_version** — an integer on mutable documents, compared on write, for the `409`
- **Reconciliation / ReconciliationLine** — statement import, matches, exceptions, closing state (the workflow the derived report grows into)
- **FxRate.source / fetched_at** — so a rate's provenance is recorded

---

## 6. Success Metrics

- All financial postings balance (debit = credit)
- Stock valuation matches GL inventory account
- 3-way match rate > 95%
- MRP net requirement accuracy: 100% exact match against independently computed expectations on the verification dataset (any difference is a defect)
- Payroll calculation error rate < 0.1%
- POS transaction latency (online) < 2 s
- Full audit trail for every master and transaction change

Added with Phases 7–15 (2026-10-10) — findings close when these hold:

- **Identity:** 100% of routes refuse an unauthenticated request (no exceptions beyond health, login, the contract and the docs); 0 routes read `X-Actor` or `X-Company-Id`
- **Isolation:** a session bound to company A is refused or returns no row for company B on 100% of routes, enumerated by a check
- **Schema:** `alembic upgrade head` from empty succeeds; `alembic check` reports 0 drift; a restore-then-upgrade drill passes
- **Background work:** 100% of declared schedules observed firing; a duplicate run produces 0 duplicate effects; 100% of runs recorded with an outcome
- **Observability:** 100% of requests produce a log line with a correlation id; readiness answers `503` within 5 s of the database going away; 0 responses contain an exception class name or message
- **Delivery:** ≥ 99% delivery success over a 100-message run with SMTP configured; 100% of failures recorded with an attempt count
- **Edge:** 0 plain-HTTP paths to the application; over-rate requests answer `429`; over-size bodies answer `413`
- **Backups:** RPO ≤ 24 h and RTO ≤ 4 h, both verified by a timed restore drill *(proposed)*
- **Numbering:** 0 gaps and 0 duplicates in 10 000 randomized allocations; 0 body-supplied numbers accepted for statutory types
- **Fiscal close:** the trial balance balances after close, retained earnings ties to the closed result, the balance sheet is unchanged by the close — 0 tolerance, any difference is a defect
- **Concurrency:** 0 lost updates under concurrent edits; 100% of mutating routes idempotent
- **Pagination:** 21 of 21 list routes paginated, each with a maximum page size
- **PII:** 0 plaintext bank account numbers in a raw table read
- **Assurance:** backend coverage ≥ 70% and frontend ≥ 60% *(proposed)*; 0 high or critical dependency findings outstanding; 0 critical accessibility violations
- **No regression:** 100% of the existing `tests/check_*.py` still pass, and the ledger-integrity gate still passes after Phases 7–14

---

## 7. Next Steps

1. Technology stack — **decided 2026-09-17**: Python (FastAPI + SQLAlchemy), Next.js, PostgreSQL, Postgres-backed job queue, PostgreSQL full-text search. The queue *library* is the only part still to pin.
2. Define detailed domain models and API contracts for Phase 1.
3. Fix the API conventions — URL versioning, error shape, pagination, auth, idempotency keys on posting endpoints — and publish the generated contract as a **versioned artifact**: with the two applications in separate repositories, that artifact is their only coupling. **API first** (decided 2026-09-17).
4. Package the stack: one container per service — database, backend, frontend (worker added with the first scheduled job) — brought up together by a single compose file that consumes each application's published image, so neither application repository needs its sibling checked out. (Decided 2026-09-17.)
5. Set up CI/CD, testing strategy (unit + integration + ledger integrity tests), building both images.
6. Create localization packs (CoA, tax, statutory) for the **Philippines** (decided 2026-09-17; previously "primary target markets", unnamed).
7. Begin Phase 1 development with GL + Inventory as the backbone.

*(All seven above landed in Phases 0–6. What follows them is §4b, Phases 7–15 — the production-readiness work the 2026-10-10 audit produced.)*

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
| Job queue | Postgres-backed, registered with the run table — **`pgqueuer`** (pinned 2026-10-10, Phase 8) |
| Authentication | Built in: email + password, Argon2id, server-side revocable sessions, TOTP MFA (decided 2026-10-10, Phase 7) |
| Deployment target | A single Linux host running Docker Compose behind Caddy with automatic TLS (decided 2026-10-10, Phase 9) |
| Compose stack home | The backend repository — where `T-0.DEPLOY.03` put it (disposed 2026-10-10) |
| Backup and recovery | Off-host `pg_dump`, retention by count and age, RPO ≤ 24 h / RTO ≤ 4 h, rehearsed quarterly (decided 2026-10-10, Phase 9) |
| Document numbering | Server-allocated, gapless, scoped to company + document type + fiscal year, gaps refused (decided 2026-10-10, Phase 10) |
| Frontend scope | The operational and admin shell only; the domain screens are their own later phases (decided 2026-10-10) |

**Still undecided:** fiscal year start (a required per-install variable; the Philippines pack's default is unconfirmed — §12 OQ-1), FX rate source, approval thresholds, 3-way-match tolerance, aging buckets, dunning levels, credit-check mode, MRP horizon and bucket, payroll cutoff, shift and overtime rules, leave accrual rules, supplier scoring weights, report schedules. §9 proposes a default for each of these; each is confirmed by the phase that needs it.

**Explicit non-goals** (decided 2026-09-17):
- Inter-company transactions and consolidation — the platform is multi-company by data isolation only
- HR beyond employee master, org chart, attendance, leave and payroll (no recruitment or appraisal)
- Manufacturing beyond BOM, routing, work centres, work orders and MRP (no PLM or quality management)
- CRM beyond the lead/opportunity pipeline and pricing rules (no marketing automation)
- Channels beyond the integrations named in §3 (no storefront, no mobile app other than offline POS)

**Explicit non-goals** (added 2026-10-10, with Phases 7–15):
- No SSO or OIDC — identity is server-owned first (recorded as a future option, not built)
- No domain screens for GL, AP, AR, procurement, inventory, manufacturing, payroll, HR or reporting — the shell that makes them possible is Phase 13; the screens are their own later phases
- No Kubernetes, no multi-region, no autoscaling — a single host is the decided target
- No Redis — the queue stays Postgres-backed
- No distributed tracing, no APM, no hosted log aggregation (only the error-tracking sink of §12 OQ-4 may be external)
- No mobile application and no rewrite of the shell as offline-first
- **No rewrite of the ledger, the posting primitive, row-level security or the audit trail.** They are the part of this platform that is already production-grade, and Phases 7–15 must not disturb them

---

## 9. Variables & Configuration

Every setting below is read once at startup and validated (§4b Phase 8, principle 6). A value marked *[proposed]* is a suggestion awaiting confirmation by the phase that adopts it; a value marked *required* has no default and refuses to start without it.

| Name | Allowed values / source | Default | Applies to |
|---|---|---|---|
| `queue_library` | `pgqueuer` | `pgqueuer` | Phase 8 |
| `worker_concurrency` | int | 4 *[proposed]* | Phase 8 |
| `log_level` | `debug`–`error` | `info` | Phase 8 |
| `log_format` | `json` \| `text` | `json` | Phase 8 |
| `metrics_enabled` | bool | `true` | Phase 8 |
| `error_tracking_dsn` | url, or unset | unset (OQ-4) | Phase 8 |
| `readiness_timeout_seconds` | 1–30 | 3 *[proposed]* | Phase 8 |
| `smtp_host` / `smtp_port` / `smtp_user` / `smtp_password` / `smtp_from` / `smtp_starttls` | host, port, credential | required for delivery | Phase 8 |
| `sms_provider` + credentials | provider enum, or unset | unset (OQ-11) | Phase 8 |
| `delivery_max_attempts` / `delivery_backoff_minutes` | int | 5 / exponential to 60 *[proposed]* | Phase 8 |
| `session_ttl_minutes` | 5–1440 | 720 *[proposed]* | Phase 7 |
| `session_idle_timeout_minutes` | 5–480 | 60 *[proposed]* | Phase 7 |
| `password_min_length` | 8–64 | 12 *[proposed]* | Phase 7 |
| `argon2_parameters` | time / memory / parallelism | library default, stated | Phase 7 |
| `lockout_threshold` / `lockout_window_minutes` | int | 5 / 15 *[proposed]* | Phase 7 |
| `mfa_required_roles` | role codes | `administrator` (OQ-5) | Phase 7 |
| `recovery_code_count` | 5–20 | 10 *[proposed]* | Phase 7 |
| `rate_limit_rps` / `rate_limit_burst` | int | 20 / 40 *[proposed]* | Phase 9 |
| `max_request_bytes` | int | 1 MiB *[proposed]* | Phase 9 |
| `field_encryption_key` (or its reference) | secret reference | required (OQ-3) | Phase 9 |
| `backup_schedule` / `backup_retention_days` | cron / int | daily 02:00 / 30 *[proposed]* | Phase 9 |
| `rpo_hours` / `rto_hours` | int | 24 / 4 *[proposed]* | Phase 9 |
| `uvicorn_workers` | int | 2 *[proposed]* | Phase 9 |
| `db_pool_size` / `db_pool_max_overflow` | int | 5 / 10 *[proposed]* | Phase 9 |
| `db_statement_timeout_ms` | int | 15000 *[proposed]* | Phase 9 |
| `tls_domain` / `acme_email` | dns / email | required (OQ-10) | Phase 9 |
| `staging_image_tag` / `production_image_tag` | registry references | required | Phase 9 |
| `doc_number_scope` | `company` \| `company_fiscal_year` | `company_fiscal_year` *[proposed]* | Phase 10 |
| `doc_number_gap_policy` | `refuse` \| `allow_recorded` | `refuse` | Phase 10 |
| `max_upload_bytes` | int | 10 MiB *[proposed]* | Phase 10 |
| `allowed_upload_types` | MIME allow-list | pdf, png, jpeg, csv, xlsx | Phase 10 |
| `fiscal_year_start_month` | 1–12, per company | **required per install** (the pack refuses to guess — OQ-1) | Phase 10 / 14 |
| `fx_rate_source` | provider enum \| `manual` | `manual` until a provider is chosen | Phase 11 |
| `reconciliation_amount_tolerance` | decimal | `0.00` | Phase 11 |
| `max_page_size` | int | 200 *[proposed]* | Phase 11 |
| `coverage_floor_backend` / `coverage_floor_frontend` | percent | 70 / 60 *[proposed]* | Phase 12 |
| `data_retention_*` (export, erasure, log, attachment) | period | pending OQ-9 | Phase 12 |

The §8 decideds stand unchanged: moving-average costing, per-company valuation scope, month period locking, no negative stock, EAN/UPC + QR, Philippines localization, `ghcr.io/paws1234`.

---

## 10. Integrations & Externals

Added by Phases 7–15; the existing boundaries (§3) keep their behaviour.

- **SMTP server** — outbound only; a failure is a recorded attempt and a retry, never a lost notification
- **SMS HTTP gateway** — outbound only; the same failure contract as SMTP
- **FX rate provider** — inbound rates; a failure leaves the last rate in force and reports staleness (provider undecided, §12 OQ-1's neighbour in §8)
- **ACME / Let's Encrypt (via Caddy)** — certificate issuance and renewal; a failure is a loud health signal, not a silent fallback to HTTP
- **Off-host backup target** — outbound dumps; an unreachable target is an alert, and the drill is the proof
- **Key material source** — read at startup, never persisted into an image or the repository
- **Error-tracking sink** — optional, outbound error events (OQ-4)
- **Biometric devices** — the pull is finally scheduled (Phase 8)
- **POS hardware** — drawer, scale, card terminal, label printer; adapters behind interfaces, exercised against simulators (OQ-8)
- **BIR / statutory filing** — outbound artefacts (Phase 10); transmission is out pending OQ-7

---

## 11. Findings Traceability — the 2026-10-10 audit → phases

| Findings | Phase | Theme |
|---|---|---|
| 1, 2, 11, 12, 29 | 7 | Identity, isolation, administration, tenant provisioning |
| 3, 4, 5, 6, 7, 8, 26, 27 | 8 | Migrations, worker, observability, delivery, health, configuration |
| 9, 10, 16, 20, 24, 25, 33, 34, 36 | 9 | TLS, rate and size limits, secrets, PII, backups, deployment, scaling, runbooks, contract release |
| 13, 14, 15, 18, 19 | 10 | Numbering, fiscal close, documents, audit read, attachments |
| 17, 32, 37, 38, 39, 40, 43 | 11 | Concurrency, pagination, reconciliation, devices, FX, idempotency, recorded ceilings |
| 21, 22, 35, 41, 42 | 12 | Lint, type and packaging, scanning, tests, export and erasure, lockfile integrity |
| 28 (shell part), 30, 31 | 13 | Operational UI, boundaries, i18n |
| 23 | 14 | Legacy onboarding |
| the audit's informational module list | 15 | Functional completeness (decision gate) |
| 28 (domain screens) | — | **Deferred by decision**: their own later phases |

---

## 12. Open Questions

These block the phase named; each needs an answer before that phase starts.

- **OQ-1 — Philippines fiscal-year start.** The pack refuses to guess and the install requires the month, so this is per-install rather than unknown; what is unconfirmed is the pack's default and the statutory calendar it implies. Blocks Phase 10's statutory artefacts and Phase 14's payroll defaults.
- **OQ-2 — Functional scope (§4b Phase 15).** Nine candidate modules, two of which contradict §8's recorded non-goals. Blocks Phases 16+.
- **OQ-3 — PII encryption key custody.** Envelope encryption with a key in a secret store, versus `pgcrypto` with a database-held key, versus a KMS-backed service. Blocks the shape of Phase 9's `app/crypto.py`.
- **OQ-4 — Error-tracking and metrics sink.** Self-hosted on the same host, or an external service reached by a DSN. Blocks Phase 8's error-tracking hook and Phase 9's alerting.
- **OQ-5 — MFA and recovery policy.** Whether MFA is mandatory for every user or only administrators, and whether recovery codes are self-service or administrator-issued. Blocks Phase 7's enforcement default.
- **OQ-6 — Machine callers.** Whether POS terminals, biometric devices or an integration partner need scoped API tokens rather than a user session. Blocks whether Phase 7 also builds tokens.
- **OQ-7 — Statutory filings.** Which BIR returns and books must be produced, in which format, and whether e-invoicing transmission is wanted. Blocks Phase 10's artefact list.
- **OQ-8 — POS hardware.** Which devices exist to integrate against, or whether adapters plus simulators are the whole deliverable. Blocks the verification half of Phase 11's POS work.
- **OQ-9 — Retention and erasure periods.** The statutory retention of financial records and the treatment of personal data inside them. Blocks Phase 12's defaults.
- **OQ-10 — DNS name, host and off-host backup target.** TLS needs a domain and an ACME contact; backups need a destination. Blocks Phase 9's TLS and the restore drill.
- **OQ-11 — SMTP/SMS provider.** Which providers, and whether staging and production share them. Blocks Phase 8's transport defaults.
- **OQ-12 — Contract publication trigger.** Whether the release job publishes on a tag automatically (recommended) or keeps a manual step with a drift check. Blocks only the mechanism in Phase 9.

---

*This plan is derived directly from the provided module specifications and prioritizes a solid double-entry financial core before layering operational modules.*
