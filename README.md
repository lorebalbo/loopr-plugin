# Loopr — Claude Code plugin

Install **Loopr** in Claude Code: the three skills (`loopr-capture`,
`loopr-resolve`, `loopr-bedtime`) **and** the hosted Loopr MCP connector, in one
install. No clone required.

```bash
# 1. add this marketplace + install the plugin
claude plugin marketplace add lorebalbo/loopr-plugin
claude plugin install loopr@loopr

# 2. authenticate the plugin's MCP server, in your browser (no key is ever pasted)
claude mcp login plugin:loopr:loopr
```

Or from inside Claude Code: `/plugin marketplace add lorebalbo/loopr-plugin`
then `/plugin install loopr@loopr`, and authenticate with `/mcp`. (A plugin's
MCP server is namespaced `plugin:<plugin>:<server>` — so it's `plugin:loopr:loopr`,
which is why `claude mcp login loopr` says "no server named loopr".)

Then just talk to Claude:

- “Add a todo to **&lt;project&gt;**: &lt;whatever you need to do&gt;” → `loopr-capture`
- “Resolve the open todos of **&lt;project&gt;**” → `loopr-resolve`
- “Schedule these for bedtime at 2am” → `loopr-bedtime`

Track everything on the board at **https://loopr-consent.onrender.com**.

## What this repo is

This is the public **distribution** repo for the Loopr Claude Code plugin —
`.claude-plugin/` (the plugin + marketplace manifests) and `skills/`. It is
generated from the Loopr backend; the MCP server, data model, and docs live in
the main Loopr project. Connecting is OAuth 2.1, and every operation is scoped
to your own account by Postgres row-level security.

## Manage

```bash
claude plugin update loopr            # update to the latest
claude plugin uninstall loopr         # remove the plugin
claude mcp logout plugin:loopr:loopr  # sign out of the MCP
```
