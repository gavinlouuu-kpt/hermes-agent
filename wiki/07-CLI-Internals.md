# 07 · CLI Internals

`hermes_cli/` contains everything the `hermes` binary does before a
conversation starts: argument parsing, config loading, auth resolution,
provider selection, the setup wizard, diagnostics, and the slash-command
registry. The conversation itself is run by `cli.py` + `run_agent.py`.

## Entry point

`pyproject.toml`:

```toml
[project.scripts]
hermes = "hermes_cli.main:main"
hermes-agent = "run_agent:main"
hermes-acp = "acp_adapter.entry:main"
```

`hermes_cli/main.py:main` dispatches on the first positional argument to a
subcommand handler.

## Subcommands

| Subcommand | File | What it does |
|---|---|---|
| `hermes` (no args) | `hermes_cli/main.py` → `cli.py` | Start interactive REPL |
| `hermes setup` | `hermes_cli/setup.py` | Interactive wizard: provider, model, API keys, messaging, terminal backend |
| `hermes model` | `hermes_cli/main.py` | Pick or change the default model |
| `hermes tools` | `hermes_cli/main.py` | Enable/disable tools and toolsets |
| `hermes config [get\|set\|edit]` | `hermes_cli/config.py` | Read/write individual config values |
| `hermes skills […]` | `hermes_cli/skills_hub.py` + `tools/skills_hub.py` | Browse/install/publish skills |
| `hermes gateway [start\|setup\|status]` | `gateway/run.py` | Messaging-gateway process |
| `hermes doctor` | `hermes_cli/doctor.py` | Diagnose Python / deps / API keys / terminals / skills |
| `hermes update` | `hermes_cli/main.py` | Self-update from PyPI or git |
| `hermes claw migrate` | `hermes_cli/claw.py` | Migrate settings from OpenClaw |
| `hermes chat <prompt>` | `hermes_cli/main.py` | Non-interactive one-shot query |

## Key modules

### `hermes_cli/config.py`

- `load_config()` reads `~/.hermes/config.yaml`, expands env-var
  references, validates against the schema, and writes atomically.
- Profile mode: `~/.hermes/profiles/<name>` for multi-workspace isolation
  (switched with `HERMES_PROFILE` or `--profile`).
- Detects managed systems (NixOS, Homebrew) so the updater doesn't try to
  rewrite read-only installs.

### `hermes_cli/auth.py`

- Resolves credentials in priority order: environment → `~/.hermes/.env`
  → `~/.hermes/auth.json` → interactive prompt.
- Handles OAuth flows for Nous Portal and any OAuth-backed providers.
- Supports multiple keys per provider (feeds `agent/credential_pool.py`).

### `hermes_cli/commands.py`

The slash-command registry. Every `/<name>` the user can type is registered
here, including the handler, help text, and autocomplete completers.
Referenced by both the CLI REPL and the gateway adapters so the command
surface is identical across interfaces.

### `hermes_cli/callbacks.py`

Terminal implementations of the agent callbacks:

- **Approval** — renders a prompt when `tools/approval.py` gates a
  dangerous command and waits for y/n.
- **Sudo** — reads a sudo password from the user securely.
- **Clarify** — asks the user a clarifying question mid-turn.
- **Streaming** — prints streamed tokens to the terminal.
- **Progress** — shows the KawaiiSpinner from `agent/display.py`.

### `hermes_cli/setup.py`

Interactive onboarding. Walks the user through:

1. Choose provider (OpenRouter, Anthropic, OpenAI, Nous Portal, Gemini,
   local, …).
2. Paste an API key (stored in `~/.hermes/.env`).
3. Pick a default model (via `hermes_cli/models.py` catalog).
4. Optional: configure messaging platforms (Telegram bot token, etc.).
5. Optional: pick a terminal backend (local / Docker / SSH / Modal).
6. Optional: import from OpenClaw if `~/.openclaw/` exists.

### `hermes_cli/doctor.py`

Diagnostics. Checks Python version, installed extras, API keys, tool
availability, terminal backends, skill manifest parseability, writable
permissions on `~/.hermes/`. Prints a coloured table.

### `hermes_cli/web_server.py`

FastAPI backend for the embedded dashboard. Serves the static bundle at
`hermes_cli/web_dist/` (built from `web/`) and exposes read/write APIs for
conversation streaming and session listing.

### `hermes_cli/skin_engine.py`

Theme system. Loads skin YAMLs from `~/.hermes/skins/`, resolves ANSI
colour codes, and overrides the default Rich theme.

## The REPL (`cli.py`)

`cli.py` wraps `prompt_toolkit` for the interactive experience:

- Multiline editing with Esc-Enter to submit
- Slash-command autocomplete backed by `hermes_cli/commands.py`
- Streaming rendering via Rich
- Session history (up/down arrows)
- Interrupt-and-redirect on Ctrl+C
- Syntax highlighting for code blocks
- Status bar with current model, tokens, and cost

On submit, it forwards the user message to an `AIAgent` instance and
renders the streamed response.

## Slash commands

See [[03-Quickstart]] for the user-facing table. Implementation-wise:

```python
# hermes_cli/commands.py (illustrative)
@register_command("model", help="Change the current model")
def cmd_model(args, ctx):
    ...
```

Completions are typed by the command author (fixed choices, file paths,
tool names, etc.).

## Configuration precedence

1. Command-line flags (`--model`, `--provider`, `--config-dir`, …)
2. Environment variables (`HERMES_*`, `OPENROUTER_API_KEY`, …)
3. `~/.hermes/config.yaml`
4. Built-in defaults (`hermes_cli/config.py`)

Per-profile config overrides the global config.

## Pitfalls

- **Don't read config directly.** Always go through
  `hermes_cli/config.load_config()` so env-var expansion and profile
  selection are honoured.
- **Don't print credentials.** The doctor uses `agent/redact.py` to mask
  keys; new subcommands should too.
- **Autocompletion is eager.** Registering a command with an expensive
  completer will slow the REPL — use cached completers.

## Source of truth

- [`hermes_cli/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/hermes_cli)
- [`cli.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli.py)
- [`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml) — `[project.scripts]`

## See also

- [[03-Quickstart]]
- [[13-UI-Surfaces]]
- [[16-Configuration]]
- [[22-Contributing]]
