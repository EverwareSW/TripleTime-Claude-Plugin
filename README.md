# TripleTime — Claude Code plugin

Control [TripleTime](https://tripletime.app) time tracking from Claude Code via slash commands and an MCP server.

## What you get

- `/tripletime <subcommand>`: list days, create, update or delete logs, create and update groups.
- An MCP server entry pointing at the TripleTime API. The AI calls semantic tools (`who-am-i`, `list-days`, `upsert-log-group`, `delete-log-group`, `upsert-log`, `delete-log`, `open-in-browser`) and has autonomy over descriptions and times.

Examples:

```
/tripletime start                     # AI infers description from your session
/tripletime start "Fixing login bug"  # explicit description
/tripletime end 17:30                 # end current log at 17:30
/tripletime track "Reviewing PR #42"  # track task, AI adds end log when done
/tripletime create-group "Standup"    # new group today
/tripletime list                      # summarize this week
```

## Install

Follow these steps to install:

1. Open Claude Code anywhere and run:
   ```
   /plugin marketplace add EverwareSW/TripleTime-Claude-Plugin
   /plugin install tripletime@tripletime
   /reload-plugins
   /tripletime login
   ```
2. Type your email and password when prompted. Claude reads your machine device name and uses `Claude Code (<device name>)` as the device_name on the token, so you can later identify and revoke it from the web UI.
3. Claude adds the token to `~/.claude/plugins/data/tripletime-tripletime/.env`.
4. Restart Claude Code so the MCP server picks up the env var.
5. Run `/mcp` — `tripletime` should be listed as connected.

## Subcommands

| Command                                             | Action                                                                                     |
|-----------------------------------------------------|--------------------------------------------------------------------------------------------|
| `/tripletime login [email]`                         | Mint a Sanctum bearer via `POST /login`. Prints env-var lines.                             |
| `/tripletime logout`                                | Remove `TRIPLETIME_TOKEN` and restart Claude Code.                                         |
| `/tripletime whoami`                                | Get current user information.                                                              |
| `/tripletime list [from] [until]`                   | Summarize log groups + logs in a date range.                                               |
| `/tripletime create-group [name] [YYYY-MM-DD]`      | Create a log group.                                                                        |
| `/tripletime update-group <id> [name] [YYYY-MM-DD]` | Update a log group.                                                                        |
| `/tripletime delete-group <id>`                     | Delete a log group and all its logs.                                                       |
| `/tripletime start [description]`                   | Start a new log. AI synthesizes description if absent.                                     |
| `/tripletime end [HH:MM]`                           | Add a log to the current log group, without a description, with the given or current time. |
| `/tripletime track [description]`                   | Track current session by creating a log and adding end log when work is done.              |
| `/tripletime end-track`                             | Stop tracking.                                                                             |
| `/tripletime create-log [field=value]`              | Create a log.                                                                              |
| `/tripletime update-log <id> [field=value]`         | Update a log.                                                                              |
| `/tripletime delete-log <id>`                       | Delete a log.                                                                              |
| `/tripletime open [from] [until]`                   | Open TripleTime in the browser.                                                            |

## Stop / end semantics

A TripleTime Log has no `end_time`. The next log's `start` *is* the previous log's end. So:

- To switch tasks: use `start` — it implicitly ends the previous log.
- To stop tracking (break, end of day): use `end` — it appends an empty-description log.

## Caveats

- 2FA-protected accounts can't mint a token via this plugin yet. Use the TripleTime web UI to mint a token manually and set `TRIPLETIME_TOKEN`.
- The token has no expiry by default. Revoke it from the web UI's account settings when no longer needed.
- This plugin only ships Claude Code support. Other MCP-aware clients (Cursor, ChatGPT, Claude Desktop) can connect to the same MCP URL — see the API repo for raw config snippets.

## Token security

`TRIPLETIME_TOKEN` is a long-lived bearer for your account. Treat it like a password:
- Don't commit it to any repo.
- Don't paste it into chat or share it with the AI in a way that lands in transcripts you publish.
- Set it via env var, not via files in this plugin.