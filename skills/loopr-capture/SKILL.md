---
name: loopr-capture
description: >-
  Capture a TODO into Loopr under a named project. Use whenever the user wants
  to add, capture, jot down, or "remember to" do a task, note a bug/feature, or
  save work for later under a project — and when editing or deleting a TODO they
  just created in this chat. Stores the user's description verbatim and generates
  the title. Triggers: "add a todo", "remind me to…", "remember to…", "note that
  we need to…", "capture this", "add to <project>", "edit/delete the last todo".
---

# Loopr — capture a TODO (FLOW.md §2)

First read `REFERENCE.md` in this skill's directory for the golden rules and the
tool cheat-sheet. Then follow the flow below.

## Adding a TODO

```
have description + project?
  no  → ask ONLY for what's missing, then continue
  yes → duplicate check (semantic, across open / in_progress / closed)
          similar found → show it, let the user decide (don't auto-skip)
        project match:
          exact         → use it
          similar (≠)   → ask "did you mean <Y>?" (stop, don't guess)
          none          → create it
        GitHub linking (skip entirely if the user says the project has NO repo):
          GitHub lookup available?  ← check ONCE per chat, cache the result
            (GitHub MCP installed, OR the `gh` CLI is present & authenticated —
             `gh auth status`; either one works, prefer the MCP if both are there)
            no  → remember to suggest installing the GitHub MCP (or `gh auth login`)
                  at the END of the flow
            yes → project.github_url already set? → skip lookup
                  else search GitHub for the repo (MCP tools, or
                       `gh repo list` / `gh search repos` / `gh repo view --json url`):
                    found     → update_project(github_url=…)
                    not found → do NOT create the repo; warn the user, continue
        generate title; add_todo(description = user text verbatim, status open)
        confirm: title only
```

Details that matter:

- **Duplicates are semantic, not textual.** Use `list_todos(project_…, query=…)`
  to pull candidates across **all three** states and judge similarity yourself.
- **Project "similar but not identical" stops and asks.** Don't silently attach
  to the closest name.
- **GitHub MCP or `gh` CLI — either works.** Repo lookup needs only one of them.
  Prefer the MCP when it's installed; otherwise, if `gh` is present and
  authenticated (`gh auth status`), use it to find the repo URL (`gh repo list`,
  `gh search repos`, `gh repo view <owner/name> --json url`). They're
  interchangeable for linking a repo.
- **GitHub lookup check is once per chat.** If the user adds several TODOs across
  messages in one session, check availability/repo wiring once and reuse it.
- **No-repo projects are local-only.** Skip the lookup entirely; such a
  project **cannot run in the cloud** (see `loopr-resolve`, cloud execution).
  Say so if cloud execution is later asked of it.
- **Repo not found ⇒ don't create it.** It probably doesn't exist; tell the user
  and proceed without `github_url`.
- Suggest installing the GitHub MCP (or running `gh auth login`) **only at the
  end**, and only if **neither** lookup method was available (it's what lets cloud
  runs clone the repo).

## Editing / deleting recent TODOs (§2.1)

Keep a reference to the TODOs created in this chat.

- "edit the last / the second-to-last / the previous TODO" → `edit_todo` on that
  one. Description stays **verbatim**; regenerate the title if the text changed.
- "delete the last one" → `delete_todo`.
- Ambiguous ("one of the last") → list the recent TODOs from this chat and ask
  **which one**.

## Tools used

`list_projects`, `create_project`, `update_project`, `list_todos`, `add_todo`,
`edit_todo`, `delete_todo`. See `REFERENCE.md` for the cheat-sheet.
