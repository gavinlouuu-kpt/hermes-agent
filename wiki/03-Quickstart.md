# 03 · Quickstart

This page assumes you've run the installer from [[02-Installation]] and have
`hermes` on your `$PATH`. In under five minutes you'll send your first
message, switch models, and see how slash commands work.

## First conversation

```bash
hermes setup    # only once — pick a provider, paste an API key
hermes          # starts the REPL
```

The REPL is a `prompt_toolkit` TUI with multiline editing, slash-command
autocomplete, and streaming tool output. The welcome banner comes from
[`hermes_cli/banner.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/banner.py).

Type a message, press **Enter**. The agent replies; if it needs to run a
tool (read a file, search the web, run a shell command) it will either
execute silently or prompt for approval, depending on the tool's risk
category (see [[17-Security-Model]]).

## Picking a model

```bash
/model                    # interactive picker — lists all configured providers
/model openrouter:anthropic/claude-sonnet-4
/model nousportal:Hermes-4-405B
```

Providers are configured in `~/.hermes/config.yaml` and backed by keys in
`~/.hermes/.env`. The picker uses `hermes_cli/models.py` for the catalog and
`agent/model_metadata.py` for context-length / capability lookup.

Also useful:

- `/reasoning` — set reasoning effort (`none`, `minimal`, `low`, `medium`,
  `high`, `xhigh`) for models that support it (Anthropic extended thinking,
  GPT-o-series, Gemini thinking).
- `/usage` — token counts and cost estimate for the current session.

## Essential slash commands

| Command | What it does |
|---|---|
| `/new` or `/reset` | Start a fresh conversation (new session ID) |
| `/model [provider:model]` | Change LLM |
| `/personality [name]` | Load a persona from `~/.hermes/personalities/` |
| `/retry` | Re-run the last assistant turn |
| `/undo` | Drop the last turn |
| `/compress` | Summarize context to free tokens |
| `/usage` | Show token usage and cost |
| `/insights [--days N]` | Weekly usage / activity summary |
| `/tools` | List enabled tools and toolsets |
| `/skills` | List available skills |
| `/<skill-name>` | Invoke a skill directly |
| `/search <query>` | FTS5 full-text search over every past session |
| `/memory` | View / edit `MEMORY.md` and `USER.md` |
| `/platforms` | List connected messaging platforms (gateway) |
| `/help` | Full command reference |

The registry behind `/` autocomplete lives in
[`hermes_cli/commands.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/commands.py).

## Switching to the messaging gateway

```bash
hermes gateway setup   # configure Telegram / Discord / Slack / …
hermes gateway start   # run the gateway (blocks)
```

After that you can DM your bot. The same slash commands work in every chat
app, routed through
[`gateway/run.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/run.py)
and the platform adapters under
[`gateway/platforms/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway/platforms).

## Your first skill invocation

Skills are reusable procedural-memory bundles. List the bundled ones:

```
/skills
```

Invoke one (names listed under `skills/*/SKILL.md`):

```
/research:arxiv  "self-improving agents"
/creative:p5js   "draw a rotating cube"
```

See [[10-Skills-System]] for authoring.

## Your first scheduled job

```
/cron add "every day at 9am" "summarize my unread github notifications"
/cron list
```

Jobs are stored under `~/.hermes/cron/` and run by the scheduler in
[`cron/scheduler.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cron/scheduler.py).
Output can be delivered to any configured messaging platform; see
[[15-Scheduling-Cron]].

## Interrupting the agent

- In the CLI: press **Ctrl+C** — the current tool call finishes but no more
  are started.
- On messaging platforms: send `/stop` or just send a new message; the
  gateway treats the new message as a course correction.

## Where to go next

- **Configure deeper** — [[16-Configuration]]
- **Understand the loop** — [[05-Architecture]] → [[06-Agent-Loop]]
- **Add a tool or skill** — [[21-Extending-Hermes]]
- **Deploy on a VPS** — [[18-Deployment]]

## Source of truth

- [`README.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/README.md) — quick-reference command tables
- [`hermes_cli/main.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/main.py) — CLI entry point
- [`hermes_cli/setup.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/setup.py) — setup wizard
- [`hermes_cli/commands.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/commands.py) — slash-command registry
- [`cli-config.yaml.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli-config.yaml.example) — annotated config template

## See also

- [[02-Installation]]
- [[07-CLI-Internals]]
- [[12-Gateway-And-Messaging]]
- [[16-Configuration]]
