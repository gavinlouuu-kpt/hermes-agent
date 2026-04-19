# 08 · Tool System

Tools are how the agent acts on the world — running shell commands, reading
files, searching the web, calling MCP servers, sending messages. Hermes uses
a **self-registering registry** pattern: each tool module calls
`registry.register(...)` at import time, and the agent discovers them
without a central dispatch table.

## Key files

- [`tools/registry.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/registry.py) — the `ToolRegistry` singleton
- [`tools/approval.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/approval.py) — dangerous-command matching and approval gating
- [`model_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/model_tools.py) — thin bridge between `AIAgent` and the registry
- [`toolsets.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/toolsets.py) — named tool groupings (web, terminal, file, research, full_stack, …)
- [`toolset_distributions.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/toolset_distributions.py) — sampling distributions used in RL

## The registry pattern

```python
# tools/my_tool.py
from .registry import registry

async def handle_my_tool(args, ctx):
    ...

registry.register(
    name="my_tool",
    schema={...},       # OpenAI-style function schema
    handler=handle_my_tool,
    toolset="productivity",
    is_available=lambda cfg: cfg.get("my_tool_enabled", True),
)
```

`discover_builtin_tools()` imports every module under `tools/` at startup
which triggers their top-level `registry.register()` calls. No manual
registration list.

## Dispatch

`model_tools.handle_function_call(name, args, ctx)` looks the tool up in
the registry and invokes its handler. It handles:

- Async/sync handlers (via a persistent per-thread event loop)
- Argument validation against the registered schema
- Result truncation to respect the context budget
- Approval gating (see below)
- Error wrapping — turning exceptions into structured `tool_result`
  messages the model can recover from

## Toolsets

A **toolset** is a named group of tools. Users enable/disable toolsets in
config; the agent includes only the enabled ones in each LLM call. Examples:

- `terminal` — shell tool + file tools + approvals
- `web` — Exa, Firecrawl, Parallel, Browser
- `research` — web + session search + memory + delegate
- `creative` — image generation + vision + file tools
- `full_stack` — everything except the research-only heavy tools

`toolsets.py` defines the default groupings. You can override per-profile
or per-platform in `config.yaml`:

```yaml
toolsets:
  default: [terminal, web, memory]
  telegram: [web, memory]      # lighter set for messaging
  cron: [terminal, memory]
```

## Bundled tools (representative)

| Tool | Purpose | Key file |
|---|---|---|
| `terminal` | Run shell commands across 6 backends | `tools/terminal_tool.py` |
| `read_file` / `write_file` / `edit_file` | File I/O with diffing | `tools/file_tools.py`, `file_operations.py` |
| `web_search` / `web_extract` | Exa + Firecrawl + Parallel | `tools/web_tools.py` |
| `browser` | Browserbase / Camofox automation | `tools/browser_tool.py` |
| `vision` | Multimodal image analysis | `tools/vision_tools.py` |
| `code_execution` | Python sandbox with RPC bridge | `tools/code_execution_tool.py` |
| `delegate` | Spawn isolated subagents | `tools/delegate_tool.py` |
| `session_search` | FTS5 search over past sessions | `tools/session_search_tool.py` |
| `memory_*` | Read/write MEMORY.md and USER.md | `tools/memory_tool.py` |
| `skills_*` | Load / create / publish skills | `tools/skills_tool.py`, `skills_hub.py` |
| `cron_*` | Manage scheduled jobs | `tools/cronjob_tools.py` |
| `mcp_*` | Invoke MCP servers | `tools/mcp_tool.py` |
| `tts` / `voice_mode` / `transcribe` | Speech I/O | `tools/tts_tool.py`, `voice_mode.py`, `transcription_tools.py` |
| `image_generation` | FLUX via FAL | `tools/image_generation_tool.py` |
| `homeassistant` | Smart-home control | `tools/homeassistant_tool.py` |
| `mixture_of_agents` | Ensemble tool | `tools/mixture_of_agents_tool.py` |
| `send_message` | Gateway message delivery | `tools/send_message_tool.py` |
| `osv_check` | Malware scan a package before install | `tools/osv_check.py` |

## Approval gating

`tools/approval.py` inspects tool-call arguments for dangerous patterns
(destructive shell commands, secret-file writes, network requests to
unexpected hosts) and either:

- Executes silently (safe calls)
- Pauses and fires the approval callback (the CLI renders a prompt, the
  gateway sends a DM)
- Rejects outright (hard-blocked operations)

Configurable allowlists live in `~/.hermes/config.yaml` under
`approval.patterns`. See [[17-Security-Model]].

## Adding a tool (short version)

1. Create `tools/my_tool.py`.
2. Define an async handler.
3. Write an OpenAI function schema (JSON Schema for args).
4. Call `registry.register(...)` at module top level.
5. Add the tool to a toolset in `toolsets.py` or via config.

Full walkthrough in [[21-Extending-Hermes]].

## How the LLM sees tools

`model_tools.get_tool_definitions()` returns a list like:

```json
[
  {
    "type": "function",
    "function": {
      "name": "terminal",
      "description": "Run a shell command in the configured backend.",
      "parameters": { ... }
    }
  },
  ...
]
```

Providers with richer tool formats (Anthropic's `tool_use` blocks, Gemini's
function calling) are handled inside the relevant `agent/*_adapter.py`.

## Pitfalls

- **Don't block the event loop.** Long-running tool work should use
  `asyncio.to_thread()` or be truly async. Synchronous CPU-heavy loops
  starve streaming callbacks.
- **Truncate large outputs.** Return at most ~4–8k tokens from a tool; long
  outputs should be summarized or paginated. `model_tools` truncates as a
  safety net but prefer explicit budgets.
- **Respect the approval gate.** New destructive tools must register a
  pattern in `approval.py` or they bypass user confirmation.
- **Keep schemas tight.** Providers vary in how strict they parse JSON
  Schema. Prefer plain `string` / `integer` / `boolean` over exotic
  constructs.

## Source of truth

- [`tools/registry.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/registry.py)
- [`tools/approval.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/approval.py)
- [`model_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/model_tools.py)
- [`toolsets.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/toolsets.py)
- [`tools/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tools) — 40+ tool modules

## See also

- [[09-Terminal-Backends]]
- [[14-MCP-Integration]]
- [[17-Security-Model]]
- [[21-Extending-Hermes]]
