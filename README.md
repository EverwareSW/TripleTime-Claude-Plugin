# TripleTime — Claude Code plugin

Control [TripleTime](https://tripletime.app) time tracking from Claude Code via slash commands and an MCP server.

## What you get

- `/tripletime <subcommand>` (with `/tt` and `/trt` aliases): start logs, end logs, create groups, list days, update or delete logs.
- An MCP server entry pointing at the TripleTime API. The AI calls semantic tools (`start_log`, `end_log`, `list_days`, …) and has autonomy over descriptions and times.

Examples:

```
/tt start                     # AI infers description from your session
/tt start "Fixing login bug"  # explicit description
/tt end 17:30                 # break log at 17:30
/tt new-group "Standup"       # new group today
/tt list                      # summarize this week
```

## Install

Add this repo as a Claude Code plugin marketplace:

```
/plugin marketplace add EverwareSW/tripletime-claude-plugin
/plugin install tripletime
```

(Replace `EverwareSW` with whatever GitHub org/user owns the published repo.)

## One-time setup

The MCP server authenticates with a Sanctum bearer token issued by the TripleTime `/login` endpoint.

1. Open Claude Code anywhere and run:
   ```
   /tripletime login your@email
   ```
2. Type your password when prompted. Claude reads your machine `hostname` and uses `<hostname> claude-code` as the device name on the token, so you can later identify and revoke it from the web UI.
3. Claude prints two lines for you to add to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):
   ```sh
   export TRIPLETIME_TOKEN="..."
   export TRIPLETIME_MCP_URL="https://doubletime-api.test/mcp"
   ```
   Use the prod URL instead of `doubletime-api.test` if you're not on the local Herd dev environment.
4. Restart Claude Code so the MCP server picks up the env var.
5. Run `/mcp` — `tripletime` should be listed as connected with 8 tools.

## Subcommands

| Command | Action |
|---|---|
| `/tripletime login [email]` | Mint a Sanctum bearer via `POST /login`. Prints env-var lines. |
| `/tripletime start [description]` | Start a new log. AI synthesizes description if absent. |
| `/tripletime end [HH:MM]` | Append a break log to today's last group. |
| `/tripletime new-group <name> [YYYY-MM-DD]` | Create a new log group. |
| `/tripletime rename-group <id> <name>` | Rename a log group. |
| `/tripletime list [from] [until]` | Summarize log groups + logs in a date range. |
| `/tripletime update <log_id> [field=value]` | Mutate description/start/marks of a log. |
| `/tripletime delete <log_id>` | Delete a log. |
| `/tripletime logout` | Remove `TRIPLETIME_TOKEN` and restart Claude Code. |

`/tt` and `/trt` are aliases.

## Stop / end semantics

A TripleTime Log has no `end_time`. The next log's `start` *is* the previous log's end. So:

- To switch tasks: use `start` — it implicitly ends the previous log.
- To stop tracking (break, end of day): use `end` — it appends an empty-description "break" log.

## Caveats

- 2FA-protected accounts can't mint a token via this plugin yet. Use the TripleTime web UI to mint a token manually and set `TRIPLETIME_TOKEN`.
- The token has no expiry by default. Revoke it from the web UI's account settings when no longer needed.
- This plugin only ships Claude Code support. Other MCP-aware clients (Cursor, ChatGPT, Claude Desktop) can connect to the same MCP URL — see the API repo for raw config snippets.

## Token security

`TRIPLETIME_TOKEN` is a long-lived bearer for your account. Treat it like a password:
- Don't commit it to any repo.
- Don't paste it into chat or share it with the AI in a way that lands in transcripts you publish.
- Set it via env var, not via files in this plugin.
