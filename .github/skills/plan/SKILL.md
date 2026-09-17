---
name: plan
description: 'Author or revise the implementation plan at .github/plan.md — goal, principles, modules, phases with exit criteria, entities, variables, metrics. Produces exactly the shape the /execute skill consumes. Use for "/plan", "write a plan", "plan this project", "plan this feature", or "revise the plan".'
argument-hint: '[what to plan]'
user-invocable: true
disable-model-invocation: true
---

# /plan — Plan Author

**Command:** `/plan [what to plan]`
**Writes:** `.github/plan.md` (the source of truth, never executed by this skill)
**This file is the skill.** Universal — not tied to any domain. Any project, feature, or system can be planned with it.

You are writing the **spec that `/execute` will later break into tasks and build**. A plan that cannot be executed without invention is a failed plan.

---

## Purpose

`/plan`:

1. Understands **what** is being planned (from the user's argument, the workspace, or a short interview)
2. Writes a structured `.github/plan.md`
3. Stops

It does **not** write code. It does **not** create `tasks.md` (that is `/execute`). It does **not** start implementing.

---

## Invocation

| User says | Agent does |
|---|---|
| `/plan` | Plan the current workspace / ongoing effort; infer scope from the repo, confirm it in one line |
| `/plan multi-warehouse inventory` | Plan that domain |
| `/plan --revise <change>` | Amend the existing `.github/plan.md` only where the change lands |
| `write a plan for X` / `plan.md` | Same as `/plan X` |

Any extra text sent with the command is the subject of the plan.

---

## Downstream contract (this is why the shape matters)

`/execute` reads this file and derives:

- `tasks.md` with ids `T-<Phase>.<Stream>.<Seq>` and gates `T-<Phase>.X.GATE`
- a **stream code per module** — so modules must be named, not implied
- **phases with their own exit criteria** — so each phase must state its exit criteria as checks
- **variables** — so anything configurable must be listed with allowed values
- **scope** — IN / OUT per phase, so out-of-scope must be explicit
- **acceptance metrics** — which only exist if you write numbers

If a heading below is absent from your plan, `/execute` will infer conservatively and possibly wrongly. **Copy the skeleton.**

---

## Repository routing (record it in the plan)

A `/do-task` run writes real code, so the plan must say **where** that code lives. Discover the workspace layout before writing, and record it in §1 of the plan:

- **Which repositories exist** — local path, remote, and what each one owns.
- **Which one holds this plan and its ledger** (the planning repository).
- **Which folders the planning repository ignores** (its `.gitignore`) — those belong to other repositories and must never be committed here.

When the workspace holds more than one application repository, **every module and every phase must be routable to exactly one of them** — the API/backend work to the backend repository, the frontend/UI work to the frontend repository. Server-side code never lands in the frontend repository and vice versa. A phase that genuinely spans both says so and names both halves.

If the plan is silent on routing, `/do-task` has to guess — which is how code ends up in the wrong repository. Record the layout even when it is a single repository; "one repository, at `<path>`" is a valid answer.

---

## Phase A — Understand the subject (before writing)

1. Restate what is being planned in one sentence. If the request is ambiguous about **what** or **for whom**, ask — once, with the options, not a questionnaire.
2. Inspect reality when the plan touches an existing system: read the codebase, configs, docs, and any existing `.github/plan.md`. A plan that ignores what already exists invents parallel work.
3. Identify **externals**: APIs, devices, vendors, people, data sources. Name them.
4. Identify **unknowns**. Do not resolve an unknown by inventing. Record it as an OPEN QUESTION that blocks a specific task, or ask.

Never invent features, modules, metrics, or constraints that were not asked for or discovered. YAGNI applies to plans too.

---

## Phase B — Write `.github/plan.md`

**Required skeleton:**

```markdown
# <Project / Feature> Plan

## 1. Project Overview

**Goal**
<one paragraph: what "done" means>

**Architecture / Approach Principles**
- <binding rule every task must obey>

**High-Level Modules**
1. <named module>
2. <named module>

**Repositories** (when the work spans more than one)
- <repo> — local path, remote, and what lands in it
- <repo> — second repository, same shape
- Planning repository: <path> — holds this plan and its ledger; ignores the app folders above

---

## 2. Module Breakdown & Scope

### 2.N <Module name>

**Core Components**
- <named thing this module owns>

**Key Capabilities**
- <what it does, stated so it can be tested>

**Scope**
- IN: <capabilities built here>
- OUT: <capabilities assigned elsewhere or never in scope>

---

## 3. Phases

### Phase N — <name>
- **Goal:** <outcome of this batch>
- **Deliverables:** <files, systems, reports, APIs, docs>
- **Dependencies:** <nothing / phase ids / externals>
- **Exit Criteria:**
  - <checkable statement, with the metric where one exists>

---

## 4. Data & Entities

- **<Entity>** — <fields/relationships that matter, and who owns it>

---

## 5. Variables & Configuration

| Name | Allowed values / source | Default | Applies to |
|---|---|---|---|
| `<name>` | <enum, range, table, tenant> | <value or "required"> | <module/phase> |

---

## 6. Integrations & Externals

- **<System>** — <what crosses the boundary, direction, and failure behaviour>

---

## 7. Success Metrics

- <metric>: <target, e.g. "> 95%", "< 2 s", "0 exceptions">

---

## 8. Non-Goals

- <explicitly out of scope, so no phase quietly grows it>

---

## 9. Open Questions

- **OQ-N:** <question> — blocks <task/phase>
```

Omit a section only when it genuinely does not apply, and say so in one line rather than deleting it silently.

### Quality rules

- **Modules are named**, not described as "the rest".
- **Every capability is testable.** "Modern UI" is not; "list screen filters by status and date range" is.
- **Exit criteria gate phases.** Each phase ends with checks, not a vibe.
- **Metrics carry numbers.** If the user gave none, propose one and mark it `[proposed]`.
- **Variables are pulled out**, not buried in prose: enums, hierarchies, thresholds/SLAs, tenancy, localization, mapping tables, and every "per X" phrase.
- **Scope is two-sided.** Write IN and OUT for each phase.
- **Dependencies are explicit**: "before", "blocked by", "requires".
- **Non-goals are written down.** This is what stops scope creep at execution time.

### Fidelity (non-negotiable)

- The user's intent is the spec. Do not add modules they did not ask for.
- Do not import assumptions from a previous plan, another project, or any example.
- If two requirements conflict, record the conflict in **Open Questions** and ask — do not pick silently.

---

## Revision mode

When `.github/plan.md` already exists:

- **Default to merge, not overwrite.** Keep every module, phase, entity, variable, constraint, and metric that is still true.
- Change only the sections the new request lands in; list what you touched.
- **Never delete a phase that `/do-task` may already have marked DONE in `tasks.md`.** If a change invalidates completed work, say so explicitly and let the user decide.
- Never rewrite `tasks.md` from here. The ledger belongs to `/execute`, and its statuses belong to `/do-task`.

---

## Failure behaviour

| Situation | Action |
|---|---|
| Request too vague to plan without invention | Ask one focused question; do not write a stub plan |
| Subject depends on an unavailable external (API keys, device, vendor doc) | Plan up to the boundary, mark it in Open Questions, and continue |
| User asks `/plan` to also implement | Write the plan, then stop and point to `/execute` |
| Workspace contradicts the request (already built) | Say what exists, plan only the delta |

---

## Output of a successful run

1. `.github/plan.md` — structured, scoped, variable-aware, with phases and exit criteria
2. The list of Open Questions that need answers
3. One line telling the user the next command: `/execute plan.md`
