---
description: Control TripleTime time tracking. Subcommands - login, logout, whoami, list, create-group, update-group, delete-group, start, end, track, create-log, update-log, delete-log, open
argument-hint: <subcommand> [args...]
---

You are operating the user's TripleTime time tracker. The MCP server `tripletime` exposes tools that map to each subcommand below. The user's input is `$ARGUMENTS` — parse the first whitespace-delimited token as the subcommand.

## Subcommands

### `login [email]`
Mint a Sanctum bearer token. Steps:
1. Find the machine device name, e.g. "Ken’s Laptop, MacBook Pro, 14-inch, 2021".
2. Build `device_name = "Claude Code <device name>"` (e.g. `Claude Code (Ken’s Laptop, MacBook Pro, 14-inch, 2021)`).
3. Ask the user for their TripleTime password (do not echo it back).
4. Resolve the API base URL: default `https://api.tripletime.app`, or use `$TRIPLETIME_API_URL` if the user has it set in their shell.
5. POST to `<base>/api/auth/login` as JSON. Use Bash `curl -sS -X POST <base>/api/auth/login -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{"email":"...","password":"...","device_name":"..."}'`.
6. Parse the `token` field from the JSON response.
7. Print to the user:
   - The authenticated user's name and email (from the users post data).
   - This line they need to add to their shell profile (e.g. `~/.zshrc`):
     ```
     export TRIPLETIME_TOKEN="<the token>"
     ```
   - Tell them to restart Claude Code so the MCP server picks up the env var.
8. Do **not** save the token to any file — only print it for the user to handle.
9. If the response is a 2FA challenge instead of a token, tell the user 2FA-protected accounts are not supported in this version and they should mint a token via the web UI.

### `logout`
Tell the user to remove `TRIPLETIME_TOKEN` from their shell profile and restart Claude Code. Note that the token remains valid server-side until they revoke it via the TripleTime web UI.

### `whoami`
Call MCP tool `who-am-i-tool`. Print the user's name, email, and any relevant metadata returned.

### `list [from] [until]`
Call MCP tool `list-days-tool` with optional `from` and `until` (YYYY-MM-DD). Summarize the result — totals per day, highlights of long blocks, anything that stands out.

### `create-group [name] [date]`
Call MCP tool `upsert-log-group-tool` **without** an `id` (creates a new group). `name` is required. `date` is YYYY-MM-DD; defaults to today.

### `update-group <id> [name] [date]`
Call MCP tool `upsert-log-group-tool` **with** the `id`. Pass any provided `name` or `date`.

### `delete-group <id>`
Call MCP tool `delete-log-group-tool`. Confirm with the user before deleting unless they said "force" or "yes".

### `start [description]`
Create a new log in the active group. Steps:
1. Call `list-days-tool` for today to identify the active log group (the last group used today; ask the user if ambiguous).
2. If no group exists today, first call `upsert-log-group-tool` to create one (prompt user for a name or infer from context).
3. Call MCP tool `upsert-log-tool` **without** an `id`, passing `log_group_id`, `start` (default: now, HH:MM 24h), and `description`.
   - If the user provided a description, use it verbatim (truncate to 255 chars).
   - If not, **synthesize one yourself** (≤ 255 chars) from the current Claude session: recent files edited, commands run, conversation topic. Be specific. Good: "Investigated MCP integration in TripleTime API", "Fixed null-pointer in LogRepository::deleteLog". Bad: "Working", "Coding".
4. Print the resulting log id, group, and description so the user can confirm.

### `end [HH:MM]`
Add an empty log (no description) to close the active log. Steps:
1. Call `list-days-tool` for today to identify the active log group.
2. Call MCP tool `upsert-log-tool` **without** an `id`, passing `log_group_id`, `start` (given HH:MM or now), and **no description**.

### `track [description]`
Track the current task continuously — creating a start log now and maintaining an end log as work progresses.
1. Call `list-days-tool` to identify the active group (create one if needed).
2. Create a log via `upsert-log-tool` with `start` = when the task began (infer from session context if possible) and the given or synthesized description.
3. When the user signals the task is done, add (or update) an empty end log via `upsert-log-tool` with `start` = finish time.
4. If the task resumes **within 5 minutes**, consider it still active — update the end log's `start` time when it finishes again.
5. If the task resumes **after 5+ minutes**, treat the gap as a break: insert an empty log first (no description, start = break start), then a new log for the resumed work with `start` = resume time.

### `create-log [field=value ...]`
Call MCP tool `upsert-log-tool` **without** an `id`. Accepted fields: `log_group_id` (required), `description`, `start` (HH:MM), `index`.

### `update-log <id> [field=value ...]`
Call MCP tool `upsert-log-tool` **with** the `id`. Pass any provided fields: `log_group_id`, `description`, `start`, `index`.

### `delete-log <id>`
Call MCP tool `delete-log-tool`. Confirm with the user before deleting unless they said "force" or "yes".

### `open [from] [until]`
Call MCP tool `open-in-browser-tool` with optional `from` and `until` (YYYY-MM-DD). After the tool returns a URL, run `open <url>` in Bash to open it in the browser.

## General rules

- You have autonomy over descriptions and times when not specified — make sensible choices, but don't invent details that aren't supported by the session.
- `upsert-log-tool` always requires `log_group_id`. If the group is unclear, call `list-days-tool` first.
- If the MCP server returns 401, tell the user the token is missing or expired and to re-run `/tripletime login`.
- If the MCP server isn't connected at all, tell the user to set `TRIPLETIME_TOKEN` in their environment and restart Claude Code.
- Stop semantics: a Log has no end_time. The next log's `start` is the previous log's end. To switch tasks use `start` (which implicitly ends the previous log). Use `end` only when stopping or breaking.