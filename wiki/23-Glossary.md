# 23 · Glossary

Definitions for the terms used throughout this wiki. Each entry links to
the page where the concept is explained in depth.

### ACP (Agent Client Protocol)
Open standard for IDE-to-agent communication, used by Zed, VS Code, and
JetBrains plugins. Implemented in
[`acp_adapter/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/acp_adapter).
See [[13-UI-Surfaces]].

### `AIAgent`
The central class that runs every conversation. Defined at
[`run_agent.py:588`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/run_agent.py#L588),
main loop `run_conversation` at line 8644. See [[06-Agent-Loop]].

### Adapter (provider adapter)
Module in `agent/` that handles provider-specific LLM API features
(Anthropic extended thinking, Bedrock auth, Gemini Cloud Code auth).
See [[05-Architecture]].

### Approval gate
The runtime check in `tools/approval.py` that pauses destructive tool
calls and asks the user for confirmation. See [[17-Security-Model]].

### Atropos
Nous Research's RL environment framework. Hermes environments inherit
from `HermesAgentBaseEnv` for Atropos compatibility. See
[[20-RL-And-Trajectories]].

### Backend (terminal backend)
One of six execution environments the `terminal` tool can use: local,
docker, ssh, modal, daytona, singularity. See [[09-Terminal-Backends]].

### Callback
Python callable passed into `AIAgent.__init__` for streaming, approval,
progress, and clarification. Lets interfaces render partial state
without blocking the loop.

### Checkpoint
Persisted state allowing a session to be resumed. Browser sessions and
some terminal backends support checkpointing via
`tools/checkpoint_manager.py`.

### Compression (context compression)
`agent/context_compressor.py` summarizes older turns when token usage
approaches the context limit. Creates a child session with
`parent_session_id`. See [[06-Agent-Loop]].

### Credential pool
`agent/credential_pool.py` — multi-key failover for an LLM provider.
Strategies: round-robin, least-used, random, fill-first.

### Cron job
A recurring, natural-language task the agent runs on a schedule. Stored
under `~/.hermes/cron/`. See [[15-Scheduling-Cron]].

### Delegate tool
`tools/delegate_tool.py` — spawns an isolated subagent (its own
`AIAgent`) for parallel workstreams. Returns only the final result,
saving the parent's context.

### Docusaurus
The static-site generator behind the canonical docs at
<https://hermes-agent.nousresearch.com/docs/>. Source in `website/`.

### FTS5
SQLite's full-text-search extension. Powers `/search` via the
`messages_fts` virtual table in `~/.hermes/state.db`. See
[[11-Memory-System]].

### Gateway
The messaging process that runs every platform adapter and feeds
incoming messages into per-session `AIAgent` instances. See
[[12-Gateway-And-Messaging]].

### Home channel
Each messaging platform can designate a default destination for
unattributed messages (cron jobs, hooks, insights). Configured per
platform (e.g. `TELEGRAM_HOME_CHANNEL`).

### Honcho
A dialectic user-modeling service by Plastic Labs; optional memory
provider plugin. See [[11-Memory-System]].

### Hook
A pluggable handler that runs on gateway startup or scheduled triggers.
`gateway/hooks.py` + `gateway/builtin_hooks/`.

### MCP (Model Context Protocol)
Open protocol for exposing tools and resources to LLM clients. Hermes is
both an MCP client (`tools/mcp_tool.py`) and an MCP server
(`mcp_serve.py`). See [[14-MCP-Integration]].

### Memory provider
Pluggable backend for cross-session memory. At most one external
provider can be active (Honcho, Mem0, RetainDB, …) plus the builtin
`MEMORY.md` / `USER.md` layer. See [[11-Memory-System]].

### `MEMORY.md` / `USER.md`
Markdown files under `~/.hermes/memories/` that hold the agent's curated
notes and a profile of the user.

### Nous Portal
Nous Research's hosted LLM endpoint (`portal.nousresearch.com`). One of
many providers configurable in `config.yaml`.

### OpenClaw
A precursor agent project; Hermes includes a migration path (`hermes
claw migrate`) to import OpenClaw settings, memories, and skills.

### Profile
Per-workspace isolation under `~/.hermes/profiles/<name>/` — separate
config, memories, skills, and session DB. Switch via `HERMES_PROFILE` or
`--profile`. See [[16-Configuration]].

### prompt_toolkit
The Python library that powers the interactive CLI REPL. See `cli.py`.

### Registry (tool registry)
The singleton in `tools/registry.py` where every tool self-registers at
import time. See [[08-Tool-System]].

### Session
A conversation identified by a UUID. Persisted in `hermes_state.db`.
May have a `parent_session_id` if created by compression.

### Skill
A reusable procedural-memory bundle (Markdown + optional scripts). See
[[10-Skills-System]].

### Skills Hub
The community registry for skills at <https://agentskills.io>. Open
standard; compatible with Hermes.

### `SOUL.md`
Optional personality file at `~/.hermes/SOUL.md` loaded into the system
prompt every session.

### Streaming
Token-by-token delivery from the LLM; surfaced to interfaces via the
streaming callback. The `prompt_toolkit` CLI renders incrementally; the
gateway batches per platform.

### System prompt
Assembled by `agent/prompt_builder.py` each turn: identity + platform
hints + skills index + context files + memory block. See
[[06-Agent-Loop]].

### Tinker
Thinking Machines Lab's RL training toolkit, optionally installed via
the `tinker-atropos` submodule. See [[20-RL-And-Trajectories]].

### Tool
An OpenAI-style function the model can call. Each is a Python handler
registered in `tools/registry.py`. See [[08-Tool-System]].

### Toolset
A named collection of tools (e.g. `web`, `terminal`, `research`,
`full_stack`) enabled by config. See [[08-Tool-System]].

### Trajectory
A recorded sequence of messages, tool calls, results, and (optionally)
rewards. Used to train tool-calling models. See
[[20-RL-And-Trajectories]].

### `uv`
The fast Python package manager (Astral) used by `setup-hermes.sh`
instead of pip.

### WSL2
Microsoft's Linux-on-Windows subsystem. Supported platform for Hermes;
native Windows is not.

## See also

- [[Home]]
- [[24-FAQ]]
