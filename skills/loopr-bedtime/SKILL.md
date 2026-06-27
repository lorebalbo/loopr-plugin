---
name: loopr-bedtime
description: >-
  Schedule a Loopr "bedtime" batch — resolve a set of TODOs overnight (or at a
  chosen start time) as scheduled cloud runs, and reschedule them later. Use when
  the user says "bedtime", "run these overnight", "schedule the todos for
  tonight/2am", "do these while I sleep", or "move my bedtime batch to <time>".
  Plans the work into waves (like loopr-resolve), schedules one client task per
  run, tags each with a recognizable marker so the whole batch can be shifted
  together. Requires cloud execution (each project needs a github_url).
---

# Loopr — bedtime scheduled batch (FLOW.md §4)

First read `REFERENCE.md` in this skill's directory for the golden rules. This
flow builds on the wave planning in `loopr-resolve` (read its "Planning — waves"
section) and on **cloud execution** — bedtime runs in the cloud, so **every
project involved must have a `github_url`** (a no-repo project is local-only;
say so and stop).

Bedtime is the one **dedicated** Loopr scheduling flow. A plain cloud run can
self-schedule for a single time without this; bedtime does more — report/
selection, wave planning, batch tagging, and bulk rescheduling.

## How scheduling works (no Loopr-side schedule store)

Loopr's data model has **no schedule table and no tag column** — the TODOs only
hold `status`. So the schedule lives in **the client's own scheduler** (Claude's
"schedule this prompt at a time" capability), not in Loopr. To make the batch
recognizable later, **embed a marker in the first line of every scheduled
prompt**:

```
LOOPR-BEDTIME | batch=<YYYY-MM-DD> | seq=<n> | wave=<w> | project=<name> | todos=<id,id,…>
```

- `batch` — the date the batch was created (one batch = one bedtime command).
- `seq` — ordinal of this run within the batch (1,2,3…), used for spacing.
- `wave` — which wave (§ planning) this run belongs to; waves are barriers.
- The rest documents what the run does.

This marker **is** the `bedtime` tag: you find your batch by listing the client's
scheduled tasks and matching the `LOOPR-BEDTIME` prefix, and you never touch a
scheduled task that lacks it.

> Capability check: this flow needs the client to (a) schedule a prompt at a
> time, and (b) **list and edit** existing scheduled tasks (for rescheduling). If
> the environment can only create schedules, you can still set a bedtime batch
> but **cannot** later shift it — tell the user that up front.

## Selecting what to run

Two modes:

1. **User specifies** the TODOs → use exactly those.
2. **You propose** → pull the candidate `open` TODOs (`list_todos`), present a
   short **report** of what you'd implement overnight, and proceed only once the
   user **approves** it.

Then **plan the waves** exactly as in `loopr-resolve` (semantic + light
file-overlap pass; dependent chains serialize, independent groups parallelize).
Each scheduled run maps to the same unit cloud execution uses: a **dependent
chain → one run**; **independent TODOs → one run each**.

## Choosing the start time

1. If the user gives a time → use it.
2. Otherwise → **2:00 (fixed default).**

Schedule the runs consistent with the wave order: a wave is a barrier, so a
later wave's runs start after the earlier wave's are expected to finish. Space
runs sensibly and record each run's intended time via `seq`.

## Creating the batch

For each run, schedule a client task whose prompt:

1. **First line** = the `LOOPR-BEDTIME | …` marker above.
2. **Body** = instructions to resolve those TODO(s) in the cloud for that project
   (clone `project.github_url`, isolated branch/PR, `in_progress → closed` per
   TODO, reconcile/merge per `loopr-resolve` §3.6).

Confirm the batch back to the user as a **list of titles + times** (titles only,
per the golden rules — never dump descriptions/IDs).

## Rescheduling — "move the start to T"

1. **List** the client's scheduled tasks; keep only those whose prompt starts
   with `LOOPR-BEDTIME` **for this batch** (match `batch=`). Never select others.
2. Find the **earliest** (`seq=1`) run's current time; compute `delta = T − that`.
3. **Shift every** bedtime run in the batch by `delta`, preserving **relative
   spacing** and **wave/dependency order** (same gaps, same sequence).
4. Re-confirm the new times.

Never move a scheduled task that isn't `LOOPR-BEDTIME`-tagged for this batch.

## Tools used

`list_todos`, `get_todo`, `edit_todo(status=…)`, `close_todo`, `list_projects`
(for `github_url`). Scheduling itself uses the **client's** scheduler, not a
Loopr tool. See `REFERENCE.md` for the cheat-sheet.
