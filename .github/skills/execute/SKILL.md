---
name: execute
description: 'Read a plan file and materialize a concrete tasks.md execution ledger from it, then stop. Writes tasks.md only: no code, no scaffolds, no plan edits. Use for "/execute", "execute the plan", "run plan.md", or turning a plan into tasks.md.'
argument-hint: '[plan-file]'
user-invocable: true
disable-model-invocation: true
---

# /execute — Plan → `tasks.md` (ledger only)

**Command:** `/execute [plan-file]`  
**Default plan:** `plan.md` (search current directory, then `artifacts/`)  
**This file is the skill.** It is **not** tied to any one domain, product, or the current ERP plan. Any well-formed plan file is valid input.

---

## Purpose

`/execute` **reads a plan and materializes the ledger from it. That is the whole job.**

It does not summarize the plan. It does not implement anything. It:

1. **Ingests** the plan
2. **Materializes** a concrete `tasks.md` — the execution ledger
3. **Stops**, and hands the run to `/do-task`

Works for software, docs, research, ops, data, design, or mixed plans.

> **LEDGER ONLY — hard rule.** `/execute` writes **exactly one file: `tasks.md`**, and then reports. No product code, no scaffolding, no folders, no docs, no config, no dependency installs, no edits to `plan.md`, no second artifact. Implementation belongs to `/do-task`, one task per run.

---

## Invocation

| User says | Agent does |
|---|---|
| `/execute` | Find `plan.md`, write `tasks.md` from it |
| `/execute plan.md` | Same, for that file |
| `/execute path/to/any-plan.md` | Same, for that file |
| `execute the plan` / `run plan.md` | Same as `/execute plan.md` |

This command has **no modes and no flags**. It always writes the whole ledger and stops. If the user passes something flag-shaped, ignore it and say there is nothing to configure.

If several `plan.md` files exist, use the one the user named. If none named, prefer `./plan.md`, then `artifacts/plan.md`, then ask which file.

**Artifacts (same directory as the plan):**

| File | Role |
|---|---|
| `plan.md` | Source of truth — **read only**, never mutate |
| `tasks.md` | **The only file this command writes** — the execution ledger |

If anything else looks necessary, say so and stop. Do not write it.

---

## Universal contract (any plan)

A plan is just structured intent. Do **not** assume ERP, accounting, sprints, or any domain. Discover structure from the file itself.

### Always extract

Scan the whole document and capture:

- **Goal / outcome** — what “done” means
- **Principles / constraints** — rules that bind every task
- **Modules / areas / workstreams** — named sections of work
- **Phases / stages / milestones** — ordered batches + their **exit criteria**
- **Deliverables** — files, systems, reports, APIs, docs
- **Entities / data / artifacts** — models, schemas, assets named in the plan
- **Variables / config / parameters** — anything selectable, localized, env-specific, or numeric
- **Scopes** — in / out of each phase or module
- **Dependencies** — “before”, “blocked by”, “requires”
- **Success metrics / acceptance** — measurable checks
- **Integrations / externals** — APIs, devices, vendors, humans
- **Non-goals** — explicitly out of scope
- **Next steps** — only if they are part of the plan, not meta-process

If a heading is missing, infer conservatively from lists and tables. **Never invent features, modules, or metrics that are not in the plan.**

### Variable extraction (universal)

A **variable** is any value the plan treats as configurable or instance-specific. Pull them from:

- Named methods / modes / flags (`FIFO`, `offline-capable`, `multi-currency`)
- Hierarchies (`Warehouse → Zone → Aisle → Bin`)
- Enumerations (`Vacation, Sick, Maternity`)
- Thresholds and SLAs (`> 95%`, `< 2 s`, `< 0.1%`)
- Localization / tenant / company / env dimensions
- Mapping tables (account mappings, tax packs, approval levels)
- “per X” phrases (per country, per item group, per customer tier)

Record each as: **name**, **allowed values / source**, **default if stated**, **where it applies**.

### Scope extraction (universal)

For every phase and module, write:

- **IN** — capabilities listed for that slice
- **OUT** — capabilities the plan assigns elsewhere, or never mentions
- **GATES** — the plan’s own exit criteria, copied verbatim then made testable

---

## Pipeline

### 1. Ingest

- Read the **entire** plan. No skimming.
- Normalize into an internal outline: goal, constraints, modules, phases, entities, variables, metrics.
- Resolve relative references inside the plan (e.g. “see section 4”).

### 2. Materialize `tasks.md`

Writing `tasks.md` **is** the output of this command. Nothing comes before it except reading the plan, and nothing comes after it except the report.

**Required skeleton (domain-agnostic):**

```markdown
# Tasks — Derived from <plan-filename>

## Source
- Plan: <path>
- Goal: <one paragraph from the plan>
- Constraints: <bullets copied/condensed from the plan>

## Variables & Scope
### Variables
- `<name>`: <meaning, domain, default>

### Global scope
- IN / OUT / NON-GOALS

### Phase scopes
- Phase N: IN / OUT / EXIT CRITERIA (verbatim from plan)

## Task ID scheme
T-<Phase>.<Stream>.<Seq>
- Phase: 0 = cross-cutting foundations that the plan says everything else depends on; then 1..N as in the plan
- Stream: short code derived from module/workstream names in the plan (do not hard-code A/I/P/M/S/H)
- Seq: zero-padded sequence
- GATE ids: T-<Phase>.X.GATE

## Repositories
- <repo> — local path, remote, state, and which tasks land in it
- Planning repository: <path> — holds the plan and this ledger
- Routing: backend/API tasks → the backend repository; frontend/UI tasks → the frontend repository; a task spanning both names both halves

## Cross-cutting / foundations
#### Task ID: T-0.<Stream>.<Seq>
...

## Phase N — <name from plan>
### Stream: <module from plan>
#### Task ID: T-N.<Stream>.<Seq>
- **Title**: action-oriented
- **Description**: exact steps
- **Scope**: boundaries (what this task does and does not do)
- **Variables / Config**: only those this task reads or writes
- **Dependencies**: task ids or externals
- **Acceptance Criteria**: testable; cite plan metrics when they exist
- **Evidence**: how completion is proven (test, file, report, screenshot)
- **Estimated Effort**: S / M / L
- **Owner Role**: inferred from the work (not from a fixed ERP roster)

### Phase Exit Gate
#### Task ID: T-N.X.GATE
- Restate the plan’s exit criteria as checks
```

**Task quality rules (any domain):**

- One task = one implementable outcome (not a slogan).
- Every named capability in the plan becomes at least one task.
- Every named entity/schema/artifact gets a create/define task before use.
- Every metric becomes acceptance criteria on the relevant task **and** on the phase gate.
- When the work spans several repositories, the ledger carries a `## Repositories` section that routes **every** task to the repository its diff lands in, and no task is left unroutable. A single-repository project still states it in one line.
- Prefer this expansion set when the plan is a build (omit categories that do not apply):

  1. Define / model
  2. Core logic / engine
  3. Interface (API, UI, CLI, doc, as applicable)
  4. Integration with the plan’s system of record
  5. Reports / outputs
  6. Configuration of extracted variables
  7. Tests / verification
  8. Phase exit gate

- If the plan is **not** software (e.g. research, ops runbook), map those eight to the equivalent artifacts (outline, gather, analyze, draft, review, publish, verify).

### 3. Stop

When `tasks.md` is written, the run is over:

1. Confirm the ledger covers the **whole** plan — every module, phase, entity, variable, metric, dependency, and gate.
2. Leave **every** task `TODO`. Do not mark `DOING` or `DONE`. Do not pre-verify anything “to be safe”.
3. Report: ledger path, task count per phase, the gates, the **first runnable id**, the **repository that first runnable id lands in** (when the ledger spans repositories), and any Open Questions or conflicts found.
4. Hand off: the next command is `/do-task` — **one task per run**.

**Do not implement, even a little.** Not `T-0.01`, not a stub, not an empty folder, not a config file, not a “quick” dependency install, not a scratch test “to check the ledger holds together”. The first line of product code written during this run is a bug in this command.

**Execution happens in `/do-task`**, which owns all status changes in `tasks.md`.

---

## Fidelity (non-negotiable)

- The plan is the spec. `tasks.md` is the work breakdown. Neither the ledger nor a later `/do-task` run may exceed the plan.
- Do not drop a module, phase, entity, variable, constraint, or metric that appears in the plan.
- Do not add a task the plan did not ask for — no “nice to have”, no refactor, no hardening pass.
- Do not import assumptions from a previous plan (including any ERP example). Each run starts from the file on disk.
- If two sections of the plan conflict, record the conflict in `tasks.md` and ask — do not pick silently.
- Localization, multi-tenant, offline, compliance, matching, valuation, etc. are **only** tasks if **this** plan names them.

---

## Idempotency (safe to re-run)

- Re-running `/execute plan.md` **updates** `tasks.md` from the current plan. Nothing else on disk changes.
- Never overwrite a `tasks.md` that has `DONE`/`DOING` items without merging: keep completed work and its recorded evidence, and add/adjust tasks for the plan diff.
- **Statuses belong to `/do-task`.** Preserve them. Never reset a `DONE` back to `TODO`, never `DOING` back to `TODO`.
- Do not mutate `plan.md`.

---

## Failure behavior

| Situation | Action |
|---|---|
| Plan file missing | Say so; do not invent a plan |
| Plan is a stub / too vague to break down without invention | Write `tasks.md` with clarifying `GATE` tasks, list the Open Questions, stop |
| Plan needs an external that is not reachable (API, device, vendor, credential) | Still write the tasks; mark those `BLOCKED` and name the missing external. Do not go set it up |
| Two sections of the plan conflict | Record the conflict in `tasks.md`, ask, do not pick silently |
| User asks `/execute` to also implement | Write the ledger, then say implementation is `/do-task` — one task per run |

---

## Minimal example (illustrative only — not a required domain)

Plan says: “Ship a CLI that converts CSV → JSON; exit when `convert --help` works and one golden file matches.”

`/execute` writes `tasks.md` with the variables (`input_encoding`, `output_indent`), the tasks (parser, CLI, golden test), and the phase gate — then **stops**. No `convert.py` exists yet. `/do-task` builds the parser; the next `/do-task` builds the CLI; the next one runs the gate.

The same procedure applies to an ERP, a website, a research memo, or any other plan.

---

## Output of a successful run

1. `tasks.md` — complete, specific, scoped, variable-aware, every task `TODO`
2. A short report: path, tasks per phase, gates, first runnable id, Open Questions
3. **Nothing else.** No source files, no other artifacts, no status changes
4. The handoff line: run `/do-task` for that first task
