# 06 · The Agent Loop

`AIAgent` is the single class that runs every conversation. This page walks
through it end-to-end with file:line pointers so you can follow along in
`run_agent.py`.

## Class: `AIAgent`

- **Definition:** [`run_agent.py:588`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/run_agent.py#L588)
- **Primary loop:** `run_conversation` at
  [`run_agent.py:8644`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/run_agent.py#L8644)

`AIAgent.__init__` wires together everything the loop needs:

- Model, provider, and `base_url` resolution
- A `credential_pool` for multi-key failover (`agent/credential_pool.py`)
- Enabled toolsets via `toolsets.py`
- The `memory_manager` (builtin memory + optional external provider)
- Callbacks for streaming, approval, progress, clarification
- A SQLite session row in `hermes_state.db`
- Checkpoint hooks if the session is resumable

## `run_conversation(user_message, system_message=None, conversation_history=None)`

```
sanitize surrogates, strip memory-context tags
     │
     ▼
memory_manager.prefetch_all()            # concurrent prefetch
     │
     ▼
_build_system_prompt()
     │  (agent/prompt_builder.py)
     │    - DEFAULT_AGENT_IDENTITY
     │    - platform hints (cli, telegram, …)
     │    - skills index (agent/skill_utils.py)
     │    - context files (SOUL.md, AGENTS.md, .cursorrules)
     │    - memory block from prefetch
     ▼
tool_definitions = model_tools.get_tool_definitions()
     │  (queries tools/registry.py for enabled toolsets)
     ▼
while True:                              # the core loop
    _run_ai_response()
      │  estimate tokens (agent/model_metadata.py)
      │  call LLM API (OpenAI SDK → base_url)
      │    - credential swap on auth errors
      │    - stream via stream_callback
      │    - extract reasoning / thinking
      │    - parse tool_calls
      │
      ▼
    if response has tool_calls:
        _execute_tools()
          │  for each tool_call:
          │    registry.dispatch(name, args, task_id)
          │    approval gate via tools/approval.py
          │    truncate result if > budget
          │    append tool_result message
          └─ continue loop
    else:
        return final response
     ▼
# post-turn
memory_manager.sync_all(user_msg, assistant_response)
_record_session()                        # SQLite append
_maybe_compress_context()                # if over threshold
memory_manager.queue_prefetch_all()
save trajectory (optional)
```

## Key helper methods on `AIAgent`

| Method | Role |
|---|---|
| `_build_system_prompt` | Compose identity + skills + memory + context files |
| `_run_ai_response` | Make the LLM API call, handle streaming, reasoning, tool calls |
| `_execute_tools` | Dispatch tool calls through the registry |
| `_handle_response` | Decide whether the turn is over |
| `_emit_status` | Fire callbacks for progress output |
| `_swap_credential` | Rotate active credential from the pool on failover |
| `_maybe_compress_context` | Trigger `agent/context_compressor.py` when over threshold |
| `_record_session` | Persist message to SQLite via `hermes_state.append_message` |

## Context compression

`agent/context_compressor.py` estimates tokens using model-specific
tokenizers (see `agent/model_metadata.py`) and summarizes older turns when
usage approaches the context window. Compression creates a **child session**
in `hermes_state.db` with a `parent_session_id` link so the original
history is never lost. Toggle via
`context_compression_enabled` / `context_compression_threshold` in
`~/.hermes/config.yaml`.

## Credential failover

[`agent/credential_pool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/credential_pool.py)
supports multiple keys per provider. Strategies:

- `round-robin` — alternate through keys
- `least-used` — pick the key with the lowest recent usage
- `random` — uniform random choice
- `fill-first` — drain one key then move to the next

Errors are classified by `agent/error_classifier.py` to distinguish
"rotate key" (auth/rate-limit) from "give up" (invalid request).

## Memory integration

`agent/memory_manager.py` orchestrates one **builtin** provider plus at
most one **external** provider (Honcho, Mem0, RetainDB, etc.). Every turn:

1. `prefetch_all()` — pull relevant context before the turn.
2. Context is wrapped in `<memory-context>` fences so injected text can't
   be confused with model output or user input.
3. After the turn, `sync_all()` records the conversation to every
   provider.
4. `sanitize_context()` strips any `<memory-context>` tags the LLM might
   have emitted to prevent memory-injection attacks.

See [[11-Memory-System]] for details.

## Provider adapters

Most providers speak OpenAI-compatible. Where they don't,
`agent/` has an adapter:

- `anthropic_adapter.py` — extended thinking, prompt caching
- `bedrock_adapter.py` — AWS SigV4 auth, Bedrock-specific params
- `gemini_cloudcode_adapter.py` — Google Cloud Code Assist

## Where to modify what

| You want to… | Touch |
|---|---|
| Change the system-prompt assembly | `agent/prompt_builder.py` |
| Add a new LLM provider with custom auth | `agent/*_adapter.py` + `agent/auxiliary_client.py` |
| Tune compression heuristics | `agent/context_compressor.py` + config |
| Change how tool calls are dispatched | `model_tools.py` + `tools/registry.py` |
| Add metrics/observability | callbacks passed to `AIAgent.__init__` |
| Adjust memory fencing | `agent/memory_manager.py` |

## Pitfalls

- **Don't add a new class for every interface.** The CLI / gateway / RL env
  all go through the same `AIAgent`. Adding parallel classes fragments the
  loop.
- **Tool handlers should be async-friendly.** `model_tools._run_async`
  keeps one event loop per thread — synchronous blocking in a tool handler
  blocks the whole turn.
- **Don't log raw messages.** Use `agent/redact.py` to scrub secrets before
  logging.

## Source of truth

- [`run_agent.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/run_agent.py) — primary file
- [`agent/prompt_builder.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/prompt_builder.py)
- [`agent/context_compressor.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/context_compressor.py)
- [`agent/credential_pool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/credential_pool.py)
- [`agent/memory_manager.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/memory_manager.py)
- [`model_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/model_tools.py)

## See also

- [[05-Architecture]]
- [[08-Tool-System]]
- [[11-Memory-System]]
- [[17-Security-Model]]
