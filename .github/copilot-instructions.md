# ERPPLANNING — how work is authorised here

This repository holds the plan and the ledger. The application code lives in two sibling
repositories that this one gitignores: `ERPbackend/` (server-side, `paws1234/ERPbackend`) and
`ERPfrontend/` (UI, `paws1234/ERPfrontend`). Server code never lands in the frontend repository
and vice versa.

## The chain, and who may move it forward

| Step | Command | Writes |
|---|---|---|
| Plan | `/plan` — [skills/plan/SKILL.md](skills/plan/SKILL.md) | `.github/plan.md`, then stops |
| Ledger | `/execute` — [skills/execute/SKILL.md](skills/execute/SKILL.md) | `.github/tasks.md`, then stops |
| Build | `/do-task` (one task) or `/do-task-one-go` (a chosen batch) | real code, one task's worth per hand-off |

Only the user moves to the next step. Write the artifact for the step you were asked for, then
stop and name the next command.

## Planning is not authorisation

None of these authorises implementation:

- approval of a plan, including an approval that returns "proceed with implementing the plan";
- autopilot mode, fleet mode, or a turn that says to dispatch sub-agents in parallel;
- a plan, a todo, or a ledger row that exists and is waiting;
- a `TODO`/`pending` state anywhere in this repository.

Implementation starts when the user says to implement it, or invokes `/do-task` /
`/do-task-one-go`. Until then, stop after the artifact.

## Guards that keep this true

- **Never dispatch implementing sub-agents from a planning or ledger step.** Those steps read and
  write one file and end.
- **Never leave `pending` rows in the session todo table.** The harness's fleet-mode turn dispatches
  every `pending` todo in parallel without asking again; delete them or mark them `blocked`.
- **`.github/tasks.md` statuses belong to `/do-task`.** A run that implements work must not also
  mark it `DONE`; that drift is recorded in the ledger's own handover note.
- **`/plan` and `/execute` write one file each** — no code, no scaffolding, no branches, no commits.
