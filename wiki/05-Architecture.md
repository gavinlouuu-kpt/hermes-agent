# 05 · Architecture

Hermes is a **layered monolith with registry-based plugins**: one agent
loop, many interfaces, one tool registry, pluggable providers for LLMs and
memory. There are no microservices — every interface (CLI, gateway, cron,
web, RL envs) ends up calling the same `AIAgent.run_conversation`.

## Mental model

```
┌─────────────────────────────────────────────────────────────┐
│  Interfaces                                                 │
│    hermes (CLI)      gateway     cron     web     RL env    │
│         │              │          │         │        │      │
│         └──────────────┴────┬─────┴─────────┴────────┘      │
│                             ▼                               │
│                    AIAgent (run_agent.py:588)               │
│                             │                               │
│     ┌───────────────────────┼───────────────────────┐       │
│     ▼                       ▼                       ▼       │
│  prompt_builder        LLM API call            tool dispatch│
│  (agent/)           (OpenAI-compatible +    (tools/registry)│
│                      provider adapter)              │       │
│                                                     ▼       │
│                                              tool handler   │
│                                          (tools/*.py)       │
│                                                             │
│  memory_manager  ─── queries → builtin + 1 external plugin  │
│  hermes_state    ─── SQLite session + FTS5 search           │
│  credential_pool ─── multi-key failover for LLM API         │
│  approval        ─── gates dangerous commands               │
└─────────────────────────────────────────────────────────────┘
```

## Lifecycle of a turn

1. **Interface receives input** — the CLI, gateway adapter, cron runner, or
   RL environment captures a user message (or a scheduled prompt).
2. **AIAgent instance is picked up or created** — the gateway keeps a
   per-session LRU cache with a 1-hour idle TTL; the CLI has one per REPL.
3. **Prefetch memory** — `memory_manager.prefetch_all()` pulls relevant
   context from the builtin store and at most one external provider.
4. **Build system prompt** — `agent/prompt_builder.py` stitches together:
   identity, platform hints, skills index, context files (`SOUL.md`,
   `AGENTS.md`, `.cursorrules`), and the fetched memory block.
5. **Get tool definitions** — `model_tools.get_tool_definitions()` queries
   `tools/registry.py` for the enabled toolsets and returns OpenAI-style
   function schemas.
6. **Call the LLM** — `_run_ai_response` invokes the OpenAI-compatible
   client (Anthropic / Bedrock / Gemini routed through their adapter in
   `agent/`). Streaming is routed via callback; credential pool handles
   failover.
7. **Dispatch any tool calls** — each `tool_call` is validated and routed
   through `registry.dispatch(tool_name, args, task_id)` which invokes the
   registered handler. `tools/approval.py` may pause and request user
   approval before execution.
8. **Append results, loop** — tool results are appended as `tool` messages;
   if the LLM returned `finish_reason == "tool_calls"`, go back to step 6.
9. **Finalize the turn** — persist the message to `hermes_state.db`, sync
   memory via `memory_manager.sync_all()`, record trajectory if enabled,
   and optionally trigger `_maybe_compress_context()`.

## Design patterns in play

- **Registry** — tools self-register at import. The registry is a singleton
  queried by `model_tools`. Adding a tool does not require editing dispatch
  code.
- **Adapter** — provider-specific features (Anthropic extended thinking,
  Bedrock signing, Gemini Cloud Code auth) live behind adapter classes in
  `agent/`.
- **Plugin** — memory providers, skills, MCP servers are all pluggable via
  entry-points-style discovery.
- **Callback** — streaming, progress, reasoning, and tool execution pump
  out through callbacks so the CLI and gateway can render partial state
  without blocking the loop.
- **LRU + TTL cache** — gateway keeps per-session `AIAgent` instances warm;
  evicts after 1 hour idle.

## Concurrency model

- **Per-thread persistent event loop.** Tool handlers may be async, but the
  agent avoids `asyncio.run()` per call — instead `model_tools._run_async`
  keeps a single long-lived loop per thread, preventing "Event loop is
  closed" errors when tools stream.
- **Gateway concurrency** comes from running each platform adapter in its
  own task; sessions are dispatched through `stream_consumer.py`.
- **Batch / RL** uses `batch_runner.py` to spawn many `AIAgent` instances
  in parallel — trajectory collection, not a shared loop.

## State surfaces

| State | Storage | Owner |
|---|---|---|
| Session messages (history, tool calls, reasoning) | SQLite + FTS5 (`~/.hermes/state.db`) | `hermes_state.py` |
| Memory (`MEMORY.md`, `USER.md`) | Markdown files under `~/.hermes/memories/` | `tools/memory_tool.py` |
| Skills | Markdown under `~/.hermes/skills/` + bundled in `skills/` | skills system |
| Cron jobs | YAML/JSON under `~/.hermes/cron/` | `cron/` |
| Config | `~/.hermes/config.yaml` | `hermes_cli/config.py` |
| Secrets | `~/.hermes/.env` | `hermes_cli/env_loader.py` |
| Auth tokens | `~/.hermes/auth.json` | `hermes_cli/auth.py` |
| Trajectories (optional) | JSON under `~/.hermes/logs/` | `agent/trajectory.py` |

## Interface-specific glue

| Interface | Bridging file |
|---|---|
| Interactive CLI | `cli.py` + `hermes_cli/main.py` |
| Messaging gateway | `gateway/run.py` |
| Cron | `cron/scheduler.py` → `AIAgent.run_conversation` |
| Web dashboard | `hermes_cli/web_server.py` (FastAPI) |
| React/Ink TUI | `tui_gateway/server.py` (Python RPC) + `ui-tui/src/gatewayClient.ts` |
| IDE (Zed/VS Code/JetBrains) | `acp_adapter/entry.py` |
| RL environments | `environments/agent_loop.py` + `environments/hermes_base_env.py` |
| Standalone MCP server | `mcp_serve.py` |
| Batch training | `batch_runner.py` |

## A labelled diagram for 06

[[06-Agent-Loop]] zooms in on steps 4–9 of the lifecycle with file:line
pointers into `run_agent.py`.

## Source of truth

- [`run_agent.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/run_agent.py) — `AIAgent` @ line 588, `run_conversation` @ line 8644
- [`agent/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/agent) — extracted internals
- [`tools/registry.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/registry.py) — dispatch
- [`AGENTS.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/AGENTS.md) — upstream architectural narrative
- [`hermes-already-has-routines.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes-already-has-routines.md) — design notes

## See also

- [[06-Agent-Loop]]
- [[08-Tool-System]]
- [[11-Memory-System]]
- [[13-UI-Surfaces]]
