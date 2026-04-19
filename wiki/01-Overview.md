# 01 · Overview

Hermes Agent is a self-improving AI assistant designed to live somewhere
persistent and be reachable from wherever you are. Point it at any LLM
provider, give it tools, and it becomes a single agent you can talk to from a
terminal, a chat app, or a scheduled cron job — with memory that carries
across sessions and skills it writes for itself when it learns something
reusable.

## Who it's for

- **Individuals** who want a personal AI running on a VPS, reachable from
  Telegram or email, with memory that lasts.
- **Developers** who want an extensible agent framework — registry-based
  tools, pluggable memory providers, platform adapters, MCP servers.
- **Researchers** who want to generate agent trajectories, train
  tool-calling models, and run benchmarks (Atropos RL integration).
- **Teams** who want a messaging-gateway bot shared across Slack or Discord
  with per-user sessions and approval gates on dangerous commands.

## Feature matrix

| Capability | Where it lives |
|---|---|
| Interactive terminal UI | [`cli.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli.py) + [`hermes_cli/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/hermes_cli) |
| Messaging gateway (13+ platforms) | [`gateway/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway) |
| 40+ tools (terminal, file, web, vision, browser, delegate, …) | [`tools/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tools) |
| 6 terminal backends (local/docker/ssh/modal/daytona/singularity) | [`tools/environments/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tools/environments) |
| Procedural memory (skills) | [`skills/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/skills), [`optional-skills/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/optional-skills) |
| Persistent memory & user modeling | [`agent/memory_manager.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/memory_manager.py), [`plugins/memory/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/plugins) |
| FTS5 session search | [`hermes_state.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_state.py), [`tools/session_search_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/session_search_tool.py) |
| Cron scheduling | [`cron/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/cron), [`tools/cronjob_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/cronjob_tools.py) |
| MCP (Model Context Protocol) client + server | [`tools/mcp_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/mcp_tool.py), [`mcp_serve.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/mcp_serve.py) |
| Web dashboard | [`web/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/web) + [`hermes_cli/web_dist/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/hermes_cli) |
| Experimental React/Ink TUI | [`ui-tui/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/ui-tui), [`tui_gateway/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tui_gateway) |
| IDE integration (ACP) | [`acp_adapter/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/acp_adapter) |
| RL trajectory generation | [`batch_runner.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/batch_runner.py), [`environments/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/environments) |

## Design philosophy

1. **One loop, many surfaces.** The CLI, gateway, cron, batch runner, and RL
   environments all call the same `AIAgent.run_conversation`. Changing the
   loop changes every interface.
2. **Provider-agnostic.** The agent speaks OpenAI-compatible. Adapters in
   `agent/anthropic_adapter.py`, `bedrock_adapter.py`, `gemini_cloudcode_adapter.py`
   handle provider-specific features (thinking, prompt caching, auth).
3. **Tools self-register.** Each file under `tools/` calls
   `registry.register(...)` at import time — you add a tool by adding a
   module, not by editing dispatch code.
4. **Everything is persistable.** Sessions → SQLite. Memory → MEMORY.md /
   USER.md plus optional external providers. Skills → Markdown/YAML.
   Trajectories → JSON. The agent can pick up where it left off.
5. **Security by approval.** Dangerous commands are matched against
   patterns in `tools/approval.py` and bounced to the user for explicit
   OK before execution.

## What's new in v0.10.0

The "Tool Gateway" release — see
[`RELEASE_v0.10.0.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/RELEASE_v0.10.0.md)
for the full change list. Prior releases are documented in
`RELEASE_v0.{2..9}.0.md`.

## Non-goals

- **Native Windows.** Install under WSL2 instead. The PowerShell installer
  in `scripts/install.ps1` exists but isn't officially supported.
- **Replacing Docusaurus.** This wiki is a distillation; the user-facing
  documentation lives at
  [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).
- **A chat frontend.** Hermes is an agent runtime. If you want a browser
  chat UI, use the embedded web dashboard, a messaging platform, or the
  ACP adapter.

## Source of truth

- [`README.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/README.md) — top-level project pitch
- [`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml) — dependencies, extras, entry points
- [`RELEASE_v0.10.0.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/RELEASE_v0.10.0.md) — most recent release notes
- [`AGENTS.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/AGENTS.md) — architectural notes for coding assistants

## See also

- [[02-Installation]]
- [[03-Quickstart]]
- [[05-Architecture]]
- [[23-Glossary]]
