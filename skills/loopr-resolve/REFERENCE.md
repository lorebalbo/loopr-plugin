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
4. **Walk the lifecycle `open → in_progress → closed` — never skip a step.**
   Before doing **any** work on a TODO — even a single one, even when you skip the
   orchestrator/wave machinery entirely and resolve it inline — your **first**
   action is `edit_todo(status='in_progress')`, before reading code or editing a
   file; mark it `closed` only when the work is actually done. This is *not* gated
   on "a wave starting": it holds for one TODO resolved inline exactly as much as
   for a planned batch. Jumping `open → closed` is a bug — the board must show work
   in progress. In a resolve flow the **orchestrator** (local, always holds the
   MCP) owns these transitions, so a cloud sub-agent lacking the MCP is never an
   excuse to skip them.
   **Review gate:** a TODO with `need_review: true` lands in **`review`** (the
   dashboard's Review column) when you close it — the server redirects the close
   and returns a `note` — and a human approves it to Done. Treat `review` as "my
   work is done, awaiting the human": report it that way and **never** try to
   force it `closed` (don't re-close it, don't clear `need_review` to get past it).
5. **When unsure, ask — don't guess.** This applies to duplicates, ambiguous
   project matches, and ambiguous "edit the last one" references.

## Tool cheat-sheet

| Intent | Tool(s) |
|---|---|
| Find/confirm a project | `list_projects` → `create_project` / `update_project` |
| Duplicate check | `list_todos(project_…, query=…)` across all statuses |
| Add a TODO | `add_todo` (`description` verbatim, `title` yours; `need_review: true` only if the user wants to review/test it before Done) |
| Link a repo | `update_project(github_url=…)` |
| Start work / finish | `edit_todo(status='in_progress')` → `close_todo` (a `need_review` TODO lands in `review` for the human) |
| Work in the user's order | `list_todos(status='open')` — returns priority high→low, then the user's `sort_order` |
| Flag for human review | `edit_todo(need_review=true)` — only when the user asks |
| See a TODO's pictures | `get_todo` → `attachments` (metadata only; images are private, viewable in the dashboard) |
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
