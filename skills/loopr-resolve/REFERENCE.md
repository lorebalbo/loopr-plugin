# Loopr — shared reference (read me first)

The three Loopr skills (`loopr-capture`, `loopr-resolve`, `loopr-bedtime`) all
rely on the rules below. Read this file before acting, then return to the skill
that was triggered. This is the single source of truth for Loopr behavior; the
skills hold only the flow-specific steps.

Loopr is a multi-user TODO orchestrator. Users capture TODOs in natural language
under named projects; **you** generate each title, plan the work by
dependencies, and resolve the TODOs — locally (Claude Code) or in the cloud
(Desktop/mobile/web), on demand or in a scheduled overnight batch.

The Loopr MCP tools are **thin data primitives**. All judgement — generating the
title, deciding whether two TODOs are duplicates, matching a "similar but not
identical" project, planning waves — lives in **you** (these skills), never in
the server.

---

## Identity & isolation

- Every call is scoped to the connected account by the server (Supabase Auth +
  Row Level Security). You never see or touch another user's data; you never
  handle keys.
- If a tool returns **not found**, treat the row as nonexistent — never infer
  that it belongs to someone else.
- `whoami` confirms the connection is scoped to the right account.

## Golden rules (never break these)

1. **`description` is the user's text, verbatim.** Never fix grammar/typos, never
   reword, never summarize. Save exactly what they wrote.
2. **You own the `title`, and only the title.** Generate a short, specific title
   from the description. Never ask the user for a title.
3. **Confirm with the title only.** After saving, recap the title — not the
   description, not internal IDs.
4. **A TODO is `closed` only when the work is actually done.** Lifecycle is
   `open → in_progress → closed`, never `closed` early.
5. **When unsure, ask — don't guess.** This applies to duplicates, ambiguous
   project matches, and ambiguous "edit the last one" references.

## Tool cheat-sheet

| Intent | Tool(s) |
|---|---|
| Find/confirm a project | `list_projects` → `create_project` / `update_project` |
| Duplicate check | `list_todos(project_…, query=…)` across all statuses |
| Add a TODO | `add_todo` (`description` verbatim, `title` yours) |
| Link a repo | `update_project(github_url=…)` |
| Start work / finish | `edit_todo(status='in_progress')` → `close_todo` |
| Edit/delete a recent TODO | `edit_todo` / `delete_todo` |
| Confirm connection identity | `whoami` |
| Review what Loopr changed | `list_audit_log` |
| Delete the whole account | `delete_account` (see below) |

Destructive tools (`delete_todo`, `delete_project`, `close_todo`) are audited
server-side — prefer `close_todo` over `delete_todo` when the user just means
"done".

**`delete_account` is irreversible** — it deletes the account and *every*
project and todo. Never call it on a casual "delete my todo". Restate exactly
what will be lost, get an explicit yes, then call it with `confirm:true`.

The server also caps request rate per user/per tool; if a tool returns HTTP 429,
back off and retry.
