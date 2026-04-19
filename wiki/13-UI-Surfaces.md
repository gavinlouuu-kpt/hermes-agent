# 13 · UI Surfaces

Hermes exposes five frontends, all talking to the same `AIAgent`:

1. **Interactive CLI** (`prompt_toolkit` — default)
2. **Messaging gateway** (13+ platforms — see [[12-Gateway-And-Messaging]])
3. **Web dashboard** (FastAPI + Vite)
4. **Experimental React/Ink TUI** (`ui-tui/` + `tui_gateway/`)
5. **IDE adapter** (Agent Client Protocol — Zed, VS Code, JetBrains)

## 1 · Interactive CLI

- Start: `hermes`
- Implementation: [`cli.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli.py) + [`hermes_cli/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/hermes_cli)
- Stack: `prompt_toolkit` for input, Rich for rendering

Features:

- Multiline editing (Esc-Enter to submit)
- Slash-command autocomplete backed by `hermes_cli/commands.py`
- Streaming tool output via callbacks
- Interrupt-and-redirect on Ctrl+C
- Session history (up/down arrows)
- Status bar showing model, token usage, cost
- Skins system for theming via `hermes_cli/skin_engine.py`

See [[07-CLI-Internals]] for the full internals.

## 2 · Messaging gateway

See [[12-Gateway-And-Messaging]]. Conceptually this is a UI surface too:
each platform renders messages differently (Telegram can edit in place,
email sends whole replies, Signal has 160-byte MMS limits).

## 3 · Web dashboard

- Backend: [`hermes_cli/web_server.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/web_server.py) (FastAPI)
- Frontend source: [`web/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/web) (Vite + Vue/TS)
- Build output: `hermes_cli/web_dist/` (packaged in the wheel)
- Requires the `web` extra: `uv pip install -e ".[web]"`

Run standalone:

```bash
hermes web        # or via docker: nousresearch/hermes-agent dashboard
```

The dashboard offers:

- Session browser (read from `hermes_state.db`)
- Live conversation streaming via WebSocket
- Token / cost usage charts
- Skill browser (installed + Skills Hub search)
- Gateway status (pulls from the health endpoint)

## 4 · Experimental React/Ink TUI

An alternate terminal UI written in React + Ink. Architecture:

```
ui-tui/ (TypeScript, Ink components)
   │
   │  JSON-RPC over stdio
   ▼
tui_gateway/ (Python RPC backend)
   │
   ▼
AIAgent (run_agent.py)
```

- [`ui-tui/src/entry.tsx`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/ui-tui/src/entry.tsx) — TTY gate, render loop
- [`ui-tui/src/app.tsx`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/ui-tui/src/app.tsx) — state machine, message handling
- [`ui-tui/src/gatewayClient.ts`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/ui-tui/src/gatewayClient.ts) — spawns Python child + RPC bridge
- [`tui_gateway/server.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tui_gateway/server.py) — RPC handlers
- [`tui_gateway/render.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tui_gateway/render.py) — Rich/ANSI rendering
- [`tui_gateway/slash_worker.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tui_gateway/slash_worker.py) — persistent CLI subprocess

Status: experimental. The `prompt_toolkit` CLI is the canonical TUI.

## 5 · IDE adapter (ACP)

[`acp_adapter/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/acp_adapter)
implements the
[Agent Client Protocol](https://github.com/zed-industries/claude-code-acp-protocol),
an open standard used by Zed, VS Code, and JetBrains to talk to any agent.

- Entry point: `hermes-acp` (from `[project.scripts]`)
- Requires the `acp` extra: `uv pip install -e ".[acp]"`
- Setup docs: [`docs/acp-setup.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/docs/acp-setup.md)

This lets you use Hermes as the agent driver inside your IDE, with
streaming responses, inline diffs, and tool-call approval dialogs rendered
by the editor.

## Parity matrix

| Feature | CLI | Gateway | Web | TUI (Ink) | ACP |
|---|---|---|---|---|---|
| Streaming responses | ✓ | platform-dependent | ✓ | ✓ | ✓ |
| Tool approval UI | ✓ | ✓ (DM prompts) | ✓ | ✓ | ✓ (editor) |
| Slash commands | ✓ | ✓ | — | ✓ | via palette |
| Voice in / out | opt-in | ✓ (Tg/WA/Discord) | — | — | — |
| Session search | `/search` | `/search` | browser UI | `/search` | via editor |
| Diff rendering | Rich | text-only | ✓ | Ink | native |

## Pitfalls

- **CLI is canonical.** New features should land in the `prompt_toolkit`
  CLI first; the Ink TUI tracks it with some lag.
- **Web dashboard is read-mostly.** Some mutations (e.g. slash commands)
  are not exposed there and still require the CLI.
- **ACP protocol version** must match your IDE extension — check
  `agent-client-protocol` in `pyproject.toml` against the editor's bundled
  version.

## Source of truth

- [`cli.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli.py)
- [`hermes_cli/web_server.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/web_server.py)
- [`web/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/web)
- [`ui-tui/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/ui-tui)
- [`tui_gateway/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tui_gateway)
- [`acp_adapter/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/acp_adapter)

## See also

- [[07-CLI-Internals]]
- [[12-Gateway-And-Messaging]]
- [[18-Deployment]]
