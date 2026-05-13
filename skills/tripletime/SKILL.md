---
name: 'tripletime'
description: Control TripleTime time tracking. Subcommands - login, logout, whoami, list, create-group, update-group, delete-group, start, end, track, create-log, update-log, delete-log, open
argument-hint: <subcommand> [args...]
disable-model-invocation: true
allowed-tools: Bash(curl *) Bash(scutil *) Bash(hostname) Bash(open *) Bash(date *) mcp__tripletime__*
---

You are operating the user's TripleTime time tracker. The MCP server `tripletime` exposes tools that map to each subcommand below. The user's input is `$ARGUMENTS` — parse the first whitespace-delimited token as the subcommand.

## Subcommands

### `login [email]`
Mint a Sanctum bearer token.
Writes `TRIPLETIME_TOKEN=...` to `${CLAUDE_PLUGIN_DATA}/.env` (~/.claude/plugins/data/tripletime-tripletime/.env). You can also write that file by hand, or set the variable in your shell environment — shell takes precedence.
Steps:
1. Get the machine name: run `scutil --get ComputerName` on macOS, or `hostname` as fallback. Trim the result.
2. Build `device_name = "Claude Code (<machine name>)"` (e.g. `Claude Code (Ken's Laptop)`).
3. Ask the user for their TripleTime password (do not echo it back).
4. Resolve the API base URL — **always** set `TRIPLETIME_URL="${TRIPLETIME_URL:-https://api.tripletime.app}"` at the top of your Bash call. Never hardcode the URL.
5. POST to `$TRIPLETIME_URL/api/auth/login` as JSON:
   ```bash
   TRIPLETIME_URL="${TRIPLETIME_URL:-https://api.tripletime.app}"
   curl -sS -X POST "$TRIPLETIME_URL/api/auth/login" -H 'Accept: application/json' -H 'Content-Type: application/json' -H "User-Agent: Claude-Code" -d '{"email":"...","password":"...","device_name":"..."}'
   ```
6. Parse the `auth_token` field from the JSON response (shape: `{"auth_token":"..."}`) and save it:
   - `mkdir -p "${CLAUDE_PLUGIN_DATA}"`.
   - Read existing `${CLAUDE_PLUGIN_DATA}/.env` if present; update/add `TRIPLETIME_TOKEN=...`, preserve other keys. Write back, no quotes around the value. Do **not** save the token anywhere else!
   - `chmod 600 "${CLAUDE_PLUGIN_DATA}/.env"` — the token is a credential.
   - Confirm, then show the status so the user sees where they stand.
7. Print to the user:
   - The authenticated user's name and email (from the response's `user` object).
   - Where the .env file was created or updated.
   - The token, show first 10 chars masked.
   - Tell the user they can also set the variable in their shell environment — shell would take precedence. E.g. add in `~/.zshrc` (remind them to source their profile (e.g. `source ~/.zshrc`) and restart claude if they set set the token manually, so the env var would become available):
     ```
     export TRIPLETIME_TOKEN="<the token>"
     ```
8. If the response is a 2FA challenge instead of a token, tell the user 2FA-protected accounts are not supported in this version and they should mint a token via the web UI.

### `logout`
1. Set `TRIPLETIME_URL="${TRIPLETIME_URL:-https://api.tripletime.app}"`. Never hardcode the URL.
2. Call `curl -sS -X POST "$TRIPLETIME_URL/api/auth/logout" -H "Authorization: Bearer $TRIPLETIME_TOKEN" -H 'Accept: application/json' -H "User-Agent: Claude-Code"` to revoke the token server-side.
3. Remove the `TRIPLETIME_TOKEN` from the `{CLAUDE_PLUGIN_DATA}/.env` (~/.claude/plugins/data/tripletime-tripletime/.env) file.
4. Tell the user to remove `TRIPLETIME_TOKEN` from their shell profile and restart claude if they set the token manually.

### `whoami`
Call MCP tool `who-am-i-tool`. Print the user's name, email, ignored_log_descriptions and any relevant metadata returned.

### `list [from] [until]`
Call MCP tool `list-days-tool` with optional `from` and `until` (YYYY-MM-DD), don't fill if not explicitly passed, defaults to this week. 
Summarize the result — totals per day, highlights of long blocks, anything that stands out.

### `create-group [name] [date]`
Call MCP tool `upsert-log-group-tool` **without** an `id` (creates a new group). `name` is optional. `date` is YYYY-MM-DD; defaults to today.

### `update-group <id> [name] [date]`
Call MCP tool `upsert-log-group-tool` **with** the `id`. Pass any provided `name` or `date`.

### `delete-group <id>`
Call MCP tool `delete-log-group-tool`. Also deletes all logs within the group! Confirm with the user before deleting unless they said "force" or "yes".

### `start [description]`
Create a new log in the active group. Steps:
1. Call `list-days-tool` for today to identify a possible usable log group (e.g. a group created for this session, a group that would be a good match for this log and without conflicting existing logs and times, the last group used today; ask the user if ambiguous).
2. If no group exists today, first call `upsert-log-group-tool` to create one (prompt user for a name or infer from context).
3. Call MCP tool `upsert-log-tool` **without** an `id`, passing `log_group_id`, `start` (default: now, HH:MM 24h), and `description`.
   - If the user provided a description, use it verbatim (truncate to 255 chars).
   - If not, **synthesize one yourself** (≤ 255 chars) from the current Claude session: Jira issue, git branch, recent files edited, commands run, conversation topic. Be specific. Good: "Investigated MCP integration in TripleTime API", "Fixed null-pointer in LogRepository::deleteLog". Bad: "Working", "Coding".
4. Print the resulting log id, group, and description so the user can confirm.

### `end [HH:MM]`
Add an empty log (no description) to close the active log. Steps:
1. If the log group of the log we are ending is known, use that. Otherwise call `list-days-tool` for today to identify the active log group.
2. Call MCP tool `upsert-log-tool` **without** an `id`, passing `log_group_id`, `start` (given HH:MM or now), and **no description**.

### `track [description]`
Track the current task continuously — creating a start log now and maintaining an end log as work progresses.

**This is a persistent loop.** Once started, steps 3–4 apply to every subsequent response for the rest of the session, unless track is ended. 
The task stays active until the user explicitly stops tracking.

1. Call `list-days-tool` to identify a suitable group and make sure there are no existing logs whos times could conflict (create a group if needed). Remember the group ID — you will need it every response. The end log ID is not yet known; it is captured after step 4.
2. Create a work log via `upsert-log-tool` with `start` = when the task began (infer from session context if possible) and the given or synthesized description. Then go to step 4 (no end log exists yet, skip step 3 this first time).
3. As your **first tool call** of every response (from the second response onwards), run `date +%H:%M` and check the gap since the end log's current `start`. If the end log ID is unknown (e.g. after session compaction), call `list-days-tool` first to re-identify it (last log with no description in the tracked group).
   - **Gap ≤ 5 min:** do nothing yet, print nothing. Go to step 4.
   - **Gap > 5 min:** leave the existing end log as-is, create a new work log with `start` = now (rounded to nearest 5 min) at index = end log's index + 2 — leaving end log's index + 1 empty, which acts as the implicit break slot. Step 4 must now create a new end log, leaving the old end log. Go to step 4.
4. As your **last tool call(s)** of every response, run `date +%H:%M` and create or update the "end log" with `start` = now (rounded to nearest 5 min). Remember its ID. Never guess the time. → Go to step 3 on the next response.

### `end-track`
Stop the active tracking loop. Steps:
1. Run `date +%H:%M` and add a final end log with `start` = now (rounded to nearest 5 min).
2. Stop the loop — do **not** apply steps 3–4 on future responses.
3. Confirm to the user that tracking has stopped and print the final end time.

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
- `upsert-log-tool` requires `log_group_id` when creating new log. If the group is unclear, call `list-days-tool` first.
- If the MCP server returns 401, tell the user the token is missing or expired and to re-run `/tripletime login`.
- If the MCP server isn't connected at all, tell the user to set `TRIPLETIME_TOKEN` in their environment and restart Claude Code.
- Stop semantics: a Log has no end_time. The next log's `start` is the previous log's end. To switch tasks use `start` (which implicitly ends the previous log). Use `end` only when stopping or breaking.
- Inserting a log at an occupied index shifts all logs at that index and above up by one.
- Changing a log's index shifts the logs between the old and new position to fill the gap: they shift down when the index increases, up when it decreases.
- Moving a log to a different group (by changing `log_group_id`) reindexes both groups and changes the computed durations of adjacent logs in each, since a log's end time is its successor's start.