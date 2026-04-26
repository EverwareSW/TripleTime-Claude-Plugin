---
description: Control TripleTime time tracking. Subcommands - login, start, end, new-group, rename-group, list, update, delete, logout
argument-hint: <subcommand> [args...]
---

You are operating the user's TripleTime time tracker. The MCP server `tripletime` exposes tools that map to each subcommand below. The user's input is `$ARGUMENTS` — parse the first whitespace-delimited token as the subcommand.

## Subcommands

### `login [email]`
Mint a Sanctum bearer token. Steps:
1. Read the machine hostname: run Bash `hostname` and trim the result.
2. Build `device_name = "<hostname> claude-code"` (e.g. `kens-macbook-pro claude-code`).
3. Ask the user for their TripleTime password (do not echo it back).
4. Resolve the API base URL: default `https://doubletime-api.test`, or use `$TRIPLETIME_API_URL` if the user has it set in their shell.
5. POST to `<base>/login` with form fields `email`, `password`, `device_name`. Use Bash `curl -sS -X POST <base>/login -H 'Accept: application/json' -d email=... -d password=... -d device_name=...`.
6. Parse the `token` field from the JSON response.
7. Print to the user:
   - The authenticated user's name and email (from the response's `user` object).
   - These two lines they need to add to their shell profile (e.g. `~/.zshrc`):
     ```
     export TRIPLETIME_TOKEN="<the token>"
     export TRIPLETIME_MCP_URL="<base>/mcp"
     ```
   - Tell them to restart Claude Code so the MCP server picks up the env var.
8. Do **not** save the token to any file — only print it for the user to handle.
9. If the response is a 2FA challenge instead of a token, tell the user 2FA-protected accounts are not supported in this version and they should mint a token via the web UI.

### `start [description]`
Call MCP tool `start_log`.
- If the user passed a description (anything after `start`), use it verbatim as the `description` arg (truncate to 255 chars).
- If they didn't pass a description, **synthesize one yourself** (≤ 255 chars) from the current Claude session: recent files edited, commands run, conversation topic. Be specific. Examples of good descriptions: "Investigated MCP integration in TripleTime API", "Fixed null-pointer in LogRepository::deleteLog". Bad: "Working", "Coding".
- For `start` time: default to now (HH:MM 24h). If the user said something like "since session start" or "since I started this session", use that earlier time.
- Print the resulting log id, group, and description so the user can confirm.

### `end [time]`
Call MCP tool `end_log`. If the user provided a time as HH:MM, pass it as `stop_time`; otherwise omit and let the server use now.

### `new-group <name> [date]`
Call MCP tool `new_log_group` with the name (required) and optional date (YYYY-MM-DD; defaults to today).

### `rename-group <id> <name>`
Call MCP tool `rename_log_group`.

### `list [from] [until]`
Call MCP tool `list_days` with optional `from` and `until` (YYYY-MM-DD). Summarize the result for the user — totals per day, highlights of long blocks, anything that stands out.

### `update <log_id> [field=value ...]`
Call MCP tool `update_log` with `log_id` and any of `description`, `start`, `marks` the user named.

### `delete <log_id>`
Call MCP tool `delete_log`. Confirm with the user before deleting unless they said something like "force" or "yes".

### `logout`
Tell the user to remove `TRIPLETIME_TOKEN` from their shell profile and restart Claude Code. Note that the token remains valid server-side until they revoke it (no MCP tool exposed for revoke yet).

## General rules

- You have autonomy over descriptions and times when not specified — make sensible choices, but don't invent details that aren't supported by the session.
- If the MCP server returns 401, tell the user the token is missing or expired and to re-run `/tripletime login`.
- If the MCP server isn't connected at all, tell the user to set `TRIPLETIME_TOKEN` in their environment and restart Claude Code.
- Stop semantics: a Log has no end_time. The next log's `start` is the previous log's end. To switch tasks use `start` (which implicitly ends the previous log). Use `end` only when stopping or breaking.
