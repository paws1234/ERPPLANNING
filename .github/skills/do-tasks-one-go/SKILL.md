---
name: do-tasks-one-go
description: 'Do a CHOSEN BATCH of tasks from tasks.md in one run: compute the selectable (unblocked, doable-now) set, group it into connected lanes, show which tasks can be selected, run the chosen ones in dependency order, then close with one commit/push/MR-PR hand-off. Same per-task discipline as /do-task (prove not-already-implemented, ladder, shortest diff, one check) but several tasks per run. Use for "/do-tasks-one-go", "do several tasks", "do a batch of the unblocked tasks", "do the connected tasks in one go", "which tasks can I run now".'
argument-hint: '[task-ids | lane] [--list] [--max N]'
user-invocable: true
disable-model-invocation: true
---

# /do-tasks-one-go — Batch Task Runner (Ponytail, lazy senior)

**Command:** `/do-tasks-one-go [task-ids | lane] [--list] [--max N]`  
**Reads:** `tasks.md` (same directory as the plan, or `artifacts/tasks.md`)  
**This file is the skill.** Universal — not tied to any domain or plan.

You are a **lazy senior developer**. Lazy means **efficient**, not careless. **The best code is the code never written.**

This command does **several tasks in one go** — the ones that are **not blocked and doable now**, and especially the **connected** ones (a chain where each task unblocks the next, or siblings that share a stream, repository and helpers). It tells the user **which tasks can be selected**, lets **them** choose, then runs that batch in dependency order and stops.

Nothing else about `/do-task` changes. Every task in the batch still gets the full discipline: proved not-already-implemented, ladder first, shortest working diff, one runnable check, repository routing, and the batch ends with **one** hand-off.

| | `/do-task` | `/do-tasks-one-go` |
|---|---|---|
| Tasks per run | exactly **one** | a **batch the user selects** |
| Who picks | the run picks the next runnable id | the run **prints the selectable set**, the **user chooses** |
| Order | — | dependency order, sequential, never parallel |
| End | one task closed, one hand-off | every task in the batch closed, **one** hand-off |
| Use when | one small step should be reviewed alone | several unblocked/connected tasks should land together |

---

## The batch rule (hard rule)

**One invocation = exactly the tasks in the selection, in dependency order, then stop.**

- The run **computes** the selectable set and **asks**; the **user selects** (*Phase B*, *Phase C*). Never auto-pick, never fill in a task the user did not choose.
- Do the **whole selection in one go** — that is the point of this command. Chaining is expected here, unlike `/do-task`.
- **Stop at the end of the selection.** Never drain the queue, never add "the next one while we are in the file", never take the rest of a lane that was not selected.
- `all` / "do everything runnable" **is** a selection — but it means exactly the set printed in *Phase B*, no more and no less.
- **No selection, no work.** An unanswered selection question starts nothing, writes nothing, touches no git (*Phase C*).
- Gates (`T-n.X.GATE`) are tasks like any other: they can be selected, they count as one task, and reaching one does not license the next phase.

---

## Where the code goes (repository routing)

Identical to `/do-task`. **Read `tasks.md`'s `## Repositories` section before writing anything** — it names each repository, its local path, and which tasks land in it.

- **Backend/API work goes to the backend repository. Frontend/UI work goes to the frontend repository.** Never the other way round.
- **Never write application code into the planning repository.** It holds the plan, the ledger and these skills — nothing else. If a task seems to need code there, stop and ask.
- A task whose scope names both halves (a client plus its API) is still **one task**: it produces one commit per repository, in the same run, and the report names both.
- **If the ledger has no `## Repositories` section, or a task cannot be routed to a repository it names — stop and ask.** Do not guess a repository, create one, move one, or add a remote.
- Run each repository's own commands (`git status`, existing-file checks, the build) with that repository as the working directory — never assume the workspace root is the repository.
- **A batch may span repositories.** That is normal: it produces **one branch per repository** for the batch, and one commit per task per repository (*Phase F*).
- Before marking anything `DONE`, confirm each changed file sits in the repository the ledger names, and report the repository with each path.

In this workspace that means: API/backend work lands in `ERPbackend`, frontend/UI work in `ERPfrontend`, and the plan, ledger and skills in the planning repository at the workspace root. The ledger's map is still the authority — if it and this line ever disagree, the ledger wins and the disagreement is a finding to report.

---

## Invocation

| User says | Agent does |
|---|---|
| `/do-tasks-one-go` | Inventory → **print the selectable set and its lanes** → ask which to run in one go |
| `/do-tasks-one-go --list` | Inventory + selectable set + lanes + blockers; **writes nothing**, starts nothing |
| `/do-tasks-one-go T-0.API.01 T-0.API.02` | Those ids, in dependency order, after proving each is runnable |
| `/do-tasks-one-go lane 2` | Every id in lane 2, in order |
| `/do-tasks-one-go all` | Every id in the selectable set printed in Phase B |
| `/do-tasks-one-go --max 3` | Cap a go at 3 tasks (default **5**); lanes longer than the cap are split |
| `do the next few tasks`, `do the unblocked ones` | Same as `/do-tasks-one-go` |

- `--list` is the read-only mode (the batch equivalent of `/do-task --status`). It prints the inventory, the selectable set, the lanes and one line per blocked id. It does **not** run the per-task already-implemented scan — that happens per task in the run.
- `--max N` bounds a go, because a batch is reviewed as one MR/PR. **An explicit id list from the user overrides the cap** — they named the tasks — but the run states the batch size it is proceeding with.
- If `tasks.md` is missing: stop. Tell the user to run `/execute plan.md` first to generate the ledger. Do **not** invent tasks.

## Outstanding hand-off — the standing question (check this first)

Unchanged from `/do-task`. **A run with an unanswered hand-off does no other work, and never cancels the question.**

1. Read `tasks.md` and look for a `## Pending hand-off` section.
2. **If it exists**: the question and its choices have been asked and are still open. **Do not ask it again** — a repeat dialog supersedes the standing one. Do not reword it, do not edit the record, do not remove it.
3. Take **no git action**, start **no task**, stop, and report the standing question and its choices verbatim: the queue is held until the user answers.
4. **When the user answers** — a chosen option, or plain text saying what to do — carry out exactly that flow, then delete the `## Pending hand-off` section as part of that run's ledger update. A `Skip` answer deletes it too.
5. **When the user is away**, nothing changes.

A batch run's unanswered ask leaves **one** record naming the **whole batch** — its ids, its repositories and branches, and the uncommitted paths (*Phase F*).

---

## Start from main (every run, before Phase A)

**Every invocation begins on an up-to-date `main`** — that is the state the ledger and the code are judged against. In **each** repository the batch will read or write:

1. Read the current branch: `git branch --show-current`
2. If it is **not** `main`, switch: `git checkout main`
3. Pull: `git pull --ff-only origin main`

The repositories a run touches are the **planning repository** (the ledger and these skills live there) and each repository the selected tasks land in, per the `## Repositories` map. `--list` runs sync too — they read the same ledger — but they write nothing.

- **Never destroy or hide work to reach `main`.** No `git stash`, no `git checkout -- .`, no `reset`, no force. Read `git status --short` first: if the tree is not clean, **stop and ask the user** — an unfinished task branch or an unanswered hand-off is the usual reason, and that diff is the user's call, not the sync's.
- `--ff-only` on purpose: if `main` has diverged, the pull fails loudly instead of quietly making a merge commit or a rebase. Report that rather than forcing it.
- State in the report which repository started on which branch (and whether the pull brought anything new).

---

## Phase A — Read the ledger (mandatory, before any code)

Read **all** of `tasks.md`. Build an inventory:

| Field | Source |
|---|---|
| Task ID | `T-<Phase>.<Stream>.<Seq>` or `T-<Phase>.X.GATE` |
| Title | task block |
| Status | `TODO` \| `DOING` \| `DONE` \| `BLOCKED` \| `SKIPPED` (default `TODO` if absent) |
| Dependencies | listed ids |
| Acceptance criteria | listed checks |
| Scope / variables | listed bounds |
| Evidence | how done is proven |
| Repository | routed via the ledger's `## Repositories` map |

Print a short inventory (counts per status, then id, status, blocked-by). Also read the ledger's `## Repositories` section: it decides which repository each task's diff lands in, and it is the only authority for that.

---

## Phase B — Compute the selectable set (the heart of this command)

This is what makes "do the connected ones in one go" real. Do it on paper, before asking.

### 1. Expand the dependency lists

Dependencies may be written as ranges (`T-2.PROC.01 … T-2.PROC.09`). Expand every range to its full id list. A task whose dependencies cannot be resolved to ids is **not** selectable — say so.

### 2. Runnable now — `R0`

A task is **runnable now** when **all** of:

- its status is `TODO` or `DOING`, **and**
- **every** dependency is `DONE` or `SKIPPED`, **and**
- no **earlier-phase Phase Exit Gate** (`T-n.X.GATE`, n < its phase) is `TODO`/`DOING`, **and**
- it can be routed to a repository the ledger names.

### 3. The batch closure — `B` (what one go could reach)

Start from `R0`, then **repeatedly add** a `TODO`/`DOING` task whose dependencies are all `DONE`/`SKIPPED` **or already in the set**. A task that only waits on another selectable task is doable in the **same go** — that is exactly the "connected" case this command exists for. Stop when nothing more can be added.

Re-apply the gate rule to the result: no task from a later phase while an earlier phase gate is not `DONE` **after** the closure.

### 4. Group `B` into lanes

A **lane** is the unit of "connected, doable in one go" — presented whole, selected whole. Build them in ledger file order, merging tasks that are connected by **any** of:

- **chain** — a dependency edge between two tasks in `B` (A unblocks B unblocks C), or B's only outstanding deps are earlier tasks in the lane
- **siblings** — same stream, same repository, adjacent in the ledger (they share helpers, conventions and the ladder's "reuse what exists" reasoning)
- **shared artifact** — the same named file, table, contract or check appears in their Evidence/Scope lines

**Cap a lane at `--max` (default 5).** A longer natural lane is **split** at the last task whose dependencies all sit inside its half, and the report says the lane continues. Prefer splitting a sibling group over splitting a chain.

### 5. Print the selectable set

| Lane | Ids (run order) | Repo | Effort | Why selectable | Unblocks next |
|---|---|---|---|---|---|
| 1 | `T-0.API.01` → `T-0.API.02` | backend → frontend | M, M | no deps | `T-1.*` API consumers |

Then, in three short lists:

- **Recommended lane** — the first lane in file order, or the longest chain starting at the first runnable id. Say why in one line.
- **Selectable but unconnected** — runnable ids that merged into no lane.
- **Not selectable** — one line per blocked id: *id — status — the exact missing dependency or external*. Never "not ready yet".

### 6. Ask which to run

Put the selection to the user with the chat question tool, naming the lanes and their ids:

- multi-select, one option per lane, plus **"Just the recommended lane"** and **"All lanes (N tasks)"**
- free text for exact ids, `lane <n>`, `all`, or `--max <n>`
- state the batch size the option implies

**Never choose for the user.** If they name an id that is not in `B`, say which dependency blocks it and ask — exclude nothing silently, and run nothing until they answer.

---

## Phase C — Wait for the selection

- **The selection is the gate.** No answer → no task started, no file written, no git action, no ledger change. Report the menu and stop.
- Nothing is recorded in `tasks.md` for an unanswered **selection** question: nothing changed, so there is no diff to hand off. The standing record is only for an unanswered **Phase F** ask (*Outstanding hand-off*).
- A selection naming a non-runnable id: report the blocker, ask whether to drop it or stop. Unanswered → stop.
- A selection larger than `--max` because the user named ids explicitly: proceed, and state the size in the report.

---

## Phase D — Run the batch (per task, the full `/do-task` discipline)

For **each** selected id, in dependency order, sequentially — never in parallel:

1. **Mark it `DOING`** in `tasks.md` before editing product code.
2. **Already implemented? (mandatory)** — read the task fully, trace the **real flow** end to end (files, functions, callers), then answer every acceptance criterion: already true in this tree, yes or no, with a pointer.
   - All true → **do not write code**: mark `DONE` with evidence and move on.
   - Some true → implement **only the missing slice**; reuse what exists.
   - None true → climb the ladder.
   - **Later tasks in a batch see the earlier tasks' diffs**, so a later task may legitimately collapse to `DONE` (already satisfied) or `SKIPPED` (covered by an earlier sibling). That is a success, not a failure — record it.
3. **The ladder** (stop at the first rung that holds): 1 YAGNI → 2 already in this codebase → 3 standard library → 4 native platform → 5 already-installed dependency → 6 one line → 7 write the minimum that works. The ladder runs **after** you understand the problem, never instead of it.
4. **Shortest working diff.** Bug-shaped work fixes the **root cause in the shared function** — grep every caller, one guard there beats one per caller. No abstractions nobody asked for, no new dependency if avoidable, deletion over addition, fewest files. Mark deliberate known-ceiling shortcuts with a **ponytail** comment naming the ceiling and the upgrade path.
5. **The check.** Non-trivial logic leaves **one** runnable check behind (an assert-based self-check or one small test file; no frameworks, no fixtures) and it is **run**. Trivial one-liners need none. A document deliverable needs none.
6. **Close the id**: acceptance criteria true with evidence, scope not exceeded, no new dependency unless the task named it, ponytails present → set `DONE` (or `BLOCKED` with the exact missing external, or `SKIPPED` with the covering path) and note the evidence in the task block.
7. **Report the id's line**, then take the next selected id.

**Never** run two tasks as one diff; never write one task's code under another's id; never let a task grow outside its ledger scope merely because the batch is large.

### Stop conditions mid-batch

Stop the batch — do not start the next id — and report, when any of these is true:

| Condition | Why it stops |
|---|---|
| The task turns out `BLOCKED` (a real external is missing) | the following selected ids may depend on it |
| An acceptance criterion needs a decision only the user can make | guessing would change the product |
| The next id is a **phase gate** and one of its checks fails | the phase is not finished; do not paper over it |
| A check or build fails and the fix is outside the task's scope | out-of-scope repairs are a new task, not a batch item |
| The selected ids are all closed | normal end → *Phase E* |

Stopping early is a normal outcome. Report the **completed** ids, the id it stopped on, the reason, and the **remaining selected** ids. Do **not** re-plan, do **not** silently substitute a different task.

---

## Phase E — Close the batch

Before the hand-off:

1. Every selected id has a final status and recorded evidence
2. Per id: scope not exceeded, the check exists where logic was non-trivial and was run, no unrequested dependency, ponytails present
3. The batch summary table: id · status · repository · files · check · evidence
4. The ids that stopped early, with the reason and what remains queued
5. Nothing outside the selection was written

---

## Phase F — Hand off (one ask for the whole batch)

**The batch ends here, exactly like a single task does.** Once *Phase E* is done and the run changed at least one file — the ledger updates alone guarantee that — the run's last act is to ask the user to **commit, push and open the MR/PR** for the whole batch, naming the **exact repositories, branches and files**, before touching git.

**The ask** — the same four choices, in this order:

1. **Commit + push + open the MR/PR** — one branch per repository for the batch, one commit per task, pushed, MR/PR opened against `main` (recommended)
2. **Commit + push** to the branch that is checked out now — no MR/PR
3. **Commit only** — leave the push to the user
4. **Skip** — leave everything uncommitted

- Consent is **per run**. An explicit "commit and push when done" in *this* run's own prompt is consent for this run. Anything said in an earlier run is **not**.
- **An unanswered ask is not an answer.** No reply → **no git action at all**: no commit, no branch, no push, no MR/PR. Report the changed paths as uncommitted and stop. Never substitute a smaller action — no "safety" commit, no parked branch, no unasked push.
- The ask then **stays outstanding, recorded**: one `## Pending hand-off` section naming the **whole batch** — its ids, each repository's state and branch, the uncommitted paths, and the choices verbatim. That record **is** the question: never re-put, never reworded, never removed, until the user answers (*Outstanding hand-off*).
- Name the request for what the remote calls it: a **pull request (PR)** on a GitHub remote, a **merge request (MR)** on a GitLab one. Never mix the names.

### Branching and commits

- **One branch per repository for the whole batch**, named `one-go/<first-task-id>` (e.g. `one-go/T-0.API.01`). The batch is reviewed as one MR/PR per repository, not one per task.
- **One commit per task, per repository**, in run order. Subject `T-<id>: <what changed>` unless the repository's own convention differs (a `CONTRIBUTING.md`, a visible history). The task id appears in the message, so each commit and its ledger evidence are findable from each other.
- The ledger update for a task belongs in the **planning repository's** commit for that task — not left floating in the working tree.
- **Stage only the files that task changed.** Never `git add -A`, `git add .` or `git commit -a`. Read `git status --short` and `git diff` in that repository first. If two tasks touched the same file, the later commit carries the later hunks — that is expected.
- The MR/PR body lists the batch: every id, its status, its evidence and its check.

### Secrets (hard rule — these remotes are public)

- Before every commit, check the staged set for `.env`, credentials, keys, tokens and passwords. If anything sensitive is staged, **stop**, unstage it, report it, and do not proceed without an explicit answer.
- A pushed secret is compromised. There is no "fix it in the next commit".

### Never

- Force-push, amend, rebase or otherwise rewrite pushed history
- `--no-verify`, a hooks-path override, or any flag that skips a check — if a hook fails, stop and report it
- Add, change or remove a remote, or push to a repository the ledger does not name
- Commit files a task did not change, or as another author
- Treat silence as consent: an unanswered ask leaves the work uncommitted
- Substitute a smaller git action for the one asked

### Report

Per repository: branch, one line per task commit (id + hash), pushed or not, and the MR/PR URL — or that the hand-off was skipped, with the paths left uncommitted. Always say the ask was put and what the answer was; "asked, not yet answered" is a complete report. Say which branch each repository was left on: the next run returns to `main` first.

---

## Status discipline

Valid statuses only: `TODO` | `DOING` | `DONE` | `BLOCKED` | `SKIPPED`

- `SKIPPED` requires a covering path or task id (YAGNI / already exists)
- `BLOCKED` requires the missing dependency or external, not a vibe
- Never mark `DONE` because the code "looks ready." Criteria + evidence.
- Gates (`T-n.X.GATE`) are `DONE` only when **every** listed exit check is true **in the tree**, not because all child tasks are marked done on paper.

---

## What this command must never do

- Invent tasks that are not in `tasks.md`, or re-derive a new plan
- **Do a task the user did not select** — no auto-picking, no "while we are in the file", no draining the queue past the selection
- Read "do several tasks" as "do everything", or read `all` as more than the set printed in Phase B
- Start any work while the selection question is unanswered
- Skip the per-task already-implemented proof, the ladder, the trace of the real flow, the one-runnable-check rule, or the repository routing — a batch is not a licence to move faster per task
- Run tasks in parallel, or fold two tasks into one diff or one commit
- Continue past a stop condition, or substitute a different task for the one that stopped
- Add "helpful" layers, folders, frameworks or config a task did not name
- Write application code into the wrong repository, into the planning repository, or into a repository the ledger does not name
- **Commit, push or open an MR/PR without the user's explicit yes in that same run** — consent is never inferred, never carried over, never assumed from silence
- Push a secret (every remote is public), `git add -A`, or force-push
- Reach `main` by throwing work away — no `stash`, `reset`, `checkout -- .`, dirty-tree checkout or forced pull

---

## Output of a successful `/do-tasks-one-go`

1. The repository/branch state it started from — `main` synced per repository, or the dirty tree it stopped and asked about
2. The inventory (counts per status) and the **selectable set**: lanes, runnable ids, recommended lane, and one line per blocked id
3. **The selection**: what was offered, and exactly what the user chose
4. Per task, in order: already-implemented verdict, ladder rung, files changed, the check and its result, final status
5. The batch summary table, plus any id that stopped early with the reason and what remains queued
6. **The hand-off, always last**: the ask — commit, push and open the MR/PR for the batch — and the user's answer, with per repository the branch, one line per task commit, the push state and the MR/PR URL; or that the ask went unanswered, the paths are left uncommitted, and it is recorded as a `## Pending hand-off` naming the whole batch
7. A closing line naming the **next** selectable id(s) — reported, **not started**
