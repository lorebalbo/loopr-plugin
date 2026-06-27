---
name: loopr-resolve
description: >-
  Resolve / work on the open TODOs of one or more Loopr projects: fetch them,
  plan the work as a dependency DAG of parallel "waves", then execute — locally
  in Claude Code (sub-agents on git worktrees) or in the cloud (cloned repo,
  isolated branches/PRs) — and reconcile/merge the results. Use when the user
  says things like "resolve the todos of X", "work on X's todos", "knock out the
  open todos in <project>", "do the UI todos only". For SCHEDULING an overnight
  batch, use loopr-bedtime instead.
---

# Loopr — resolve TODOs (FLOW.md §3)

First read `REFERENCE.md` in this skill's directory for the golden rules and the
tool cheat-sheet. Then follow the flow.

Hierarchy: **home orchestrator → per-project orchestrator → per-TODO sub-agent.**

## Home orchestrator — local only (§3.1)

"Resolve the TODOs of X and Y" (one or many projects). For each project:

1. Find its working directory:
   - **cache** `~/.todos/projects.json` (`{ "<project name>": "<abs path>" }`,
     stays local, never on Supabase), else
   - **auto-discovery** over dev roots (e.g. `~/Developer`): match a repo whose
     git `remote` equals `project.github_url`.
   - still not found → **ask the user** where it is (then cache it).
2. Spawn a per-project orchestrator scoped to that directory.

## Per-project orchestrator (§3.2)

1. Query the project's `open` TODOs — apply any **filter** ("UI only", "detail Y
   only") to that query (`list_todos(project_…, status='open', query=…)`).
2. **Plan** the work (waves, below).
3. Execute: **local** if running in Claude Code, **cloud** if on
   Desktop/mobile/web.

**Status is the orchestrator's job (required).** The per-project orchestrator
runs locally and always holds the MCP, so *it* — not the sub-agents — drives
every TODO's status. **Before any work touches a TODO, its status is already
`in_progress`.** This is unconditional and is *not* gated on "a wave starting":
even when the ask is a single TODO you resolve inline — no home/per-project
orchestrator, no waves — your **first** MCP call is still
`edit_todo(status='in_progress')`, before reading code or editing a file. In the
batched flow the orchestrator does this for **every** TODO in a wave *before* any
sub-agent or cloud run begins. It calls `close_todo` for a TODO only once that
TODO's work is complete and merged/verified (§3.6). A TODO must never go
`open → closed` directly — `in_progress` is written at pickup (a cloud sub-agent
may not even have the MCP, which is exactly why the orchestrator owns this).
Grouping decides *who does the work* — one sub-agent can resolve several
grouped/sequenced TODOs in plan order — not who sets status.

## Planning — waves (§3.3)

Build a dependency **DAG** from two signals:

1. **Semantic** — each TODO's title/description.
2. **Code-aware (light pass)** — a quick repo scan estimating which files/modules
   each TODO touches. **File overlap is the key signal:** TODOs hitting the same
   files are **not** safely parallel (cloud = branch conflicts) → serialize them.
   No deep analysis; **when in doubt, serialize.**

Output = **waves**: each wave is a set of independent TODOs run in parallel; the
next wave starts **only after** the previous completes (waves are barriers).

- **Max concurrency is user-configurable, default 7.** Independent groups beyond
  the limit queue and start as slots free up.

Example: `users` table (1); `POST /login` (2, dep 1); `GET /profile` (3, dep 1);
README (4, independent). → Wave 1: {1, 4}; Wave 2: {2, 3}. If 2 and 3 touched the
same file, serialize them into separate waves.

## Local execution (§3.4)

Apply the plan with sub-agents: independent groups in **parallel** (background
sub-agents on separate **git worktrees**), dependent chains in **sequence**.

**Repo guardrail:** before working, compare the current dir's git `remote` with
`project.github_url`. Mismatch → an **overridable warning** ("this may not be the
same repo"), not a hard block.

## Cloud execution (§3.5)

No local filesystem: the repo is **cloned** from `project.github_url`.

- **Requires `project.github_url`.** A no-repo project is local-only — say so up
  front and stop.
- **Dependent chain →** bundle into **one prompt**, in order → **1 run, 1 branch**.
- **Independent TODOs →** **parallel runs**, **1 branch each** (`run ↔ branch ↔ PR`).
- A run can fire **now** or be **scheduled** — both are standard client features,
  no custom command. (The overnight batch is different — see `loopr-bedtime`.)

## Merge & reconciliation (§3.6)

The file-overlap estimate is light and can be wrong: two "independent" agents may
have touched the same files. After execution:

- Merge in **deterministic order**, following the plan's waves.
- Clean merge → integrate directly.
- Conflict on shared files → **never blind-overwrite**: open the conflict and
  **combine** both agents' changes (optionally a dedicated merge sub-agent),
  preserving everyone's work.
- Then **verify** the merged result (build/tests if available).

Same idea both environments: worktrees locally, branch/PR in the cloud (PRs
touching the same files are reconciled, not blindly auto-merged).

## Tools used

`list_todos`, `get_todo`, `edit_todo(status=…)`, `close_todo`, `list_projects`.
See `REFERENCE.md` for the cheat-sheet.
