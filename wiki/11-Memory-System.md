# 11 · Memory System

Hermes has three layers of memory: **per-session history** (SQLite), a
**builtin curated memory** (`MEMORY.md` + `USER.md` Markdown files), and
optional **external providers** (Honcho, Mem0, RetainDB, etc.). All three
are stitched together by `agent/memory_manager.py` and injected into every
turn through fenced context blocks.

## Layer 1 — session history (SQLite)

Every message — user, assistant, tool call, tool result — is persisted to
SQLite at `~/.hermes/state.db` by
[`hermes_state.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_state.py).

Schema highlights:

- `sessions` — id, source (cli / telegram / cron / …), model,
  message_count, cost, parent_session_id (for compression links)
- `messages` — session_id, role, content, tool_calls, reasoning, timestamp
- `messages_fts` — FTS5 virtual table; full-text index over message
  content

Read via `tools/session_search_tool.py`:

```
/search "telemetry dashboard"
```

Returns matching sessions, then the agent summarizes the result with an
auxiliary client call (`agent/auxiliary_client.py`).

## Layer 2 — builtin curated memory

Two Markdown files under `~/.hermes/memories/`:

- **`MEMORY.md`** — agent notes: "the user prefers Poetry for Python",
  "project X lives at ~/code/widget-factory", etc.
- **`USER.md`** — a user profile: name, pronouns, preferences, ongoing
  projects.

Read/write via [`tools/memory_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/memory_tool.py).
The agent nudges itself to update these files periodically — a closed
loop where today's conversation informs tomorrow's system prompt.

Also: **`SOUL.md`** at `~/.hermes/SOUL.md` — an optional personality file
loaded into the system prompt every session. Think of it as the agent's
character sheet.

## Layer 3 — external memory providers

At most **one** external provider can be active at a time, configured in
`~/.hermes/config.yaml`:

```yaml
memory:
  provider: honcho      # builtin | holographic | honcho | hindsight | supermemory | mem0 | retaindb | openviking | byterover
  honcho:
    api_key: ${HONCHO_API_KEY}
    workspace_id: ...
```

Plugins live under
[`plugins/memory/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/plugins).
Each implements a common interface (`prefetch`, `sync`, `search`).

Available providers:

| Provider | What it does |
|---|---|
| `builtin` | Just MEMORY.md + USER.md (no external service) |
| `holographic` | Local SQLite + FTS5 with holographic reduced representations |
| `honcho` | Cross-session user modeling, dialectic reasoning ([Plastic Labs](https://github.com/plastic-labs/honcho)) |
| `hindsight` | Knowledge graph, entity resolution |
| `supermemory` | Semantic memory service |
| `mem0` | Server-side LLM fact extraction |
| `retaindb` | Hybrid search (vector + BM25 + reranking) |
| `openviking` | ByteDance filesystem-style hierarchical memory |
| `byterover` | Hierarchical knowledge tree with `brv` CLI |

## MemoryManager orchestration

`agent/memory_manager.py` runs every turn:

1. **`prefetch_all()`** — concurrently query all active providers for
   relevant context.
2. **`build_memory_context_block()`** — wrap results in
   `<memory-context>...</memory-context>` fences.
3. **Inject** into the system prompt through `agent/prompt_builder.py`.
4. After the turn, **`sync_all()`** records user/assistant messages to
   every provider.
5. **`sanitize_context()`** strips any `<memory-context>` tags the LLM
   might have emitted (defence against prompt-injection via memory
   recall).
6. **`queue_prefetch_all()`** kicks off the next prefetch in the
   background so it's warm for the next turn.

## Context fencing

Memory-injected text is always wrapped in unmistakable fences:

```
<memory-context provider="honcho">
  User prefers concise answers. Last project: widget-factory.
</memory-context>
```

The model is instructed in the system prompt to treat these as context
**about** the conversation, not as content **from** the user. Fences are
stripped from any model output before persistence.

## FTS5 session search

`tools/session_search_tool.py` exposes:

```
session_search(query: str, days: int = 90)
→ [{session_id, snippet, score, timestamp}, ...]
```

The `messages_fts` virtual table is populated via SQLite triggers on
INSERT. Results are summarized with a small auxiliary LLM call so the
agent can recall "what did we decide last Tuesday about X?" cheaply.

## Compression links

When context compression fires (see [[06-Agent-Loop]]), Hermes creates a
**child session** in `hermes_state.db` with a `parent_session_id` pointing
to the original. Full history is preserved; the model only sees the
compressed summary plus recent turns.

## Pitfalls

- **Memory bloat.** If `MEMORY.md` grows past a few KB the prompt gets
  expensive. Use the `/memory cleanup` command or let the agent curate.
- **One external provider.** Mixing two external providers is not
  supported — they'd both write on every turn and can disagree.
- **Fence stripping.** Custom memory plugins must output text without
  embedding `<memory-context>` tags; the sanitizer strips them aggressively.
- **Privacy.** External providers ship your conversations to a third-party
  service. For sensitive use stick with `builtin` or `holographic`.

## Source of truth

- [`hermes_state.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_state.py)
- [`agent/memory_manager.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/memory_manager.py)
- [`tools/memory_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/memory_tool.py)
- [`tools/session_search_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/session_search_tool.py)
- [`plugins/memory/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/plugins)
- [`docs/honcho-integration-spec.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/docs/honcho-integration-spec.md)

## See also

- [[06-Agent-Loop]]
- [[10-Skills-System]]
- [[17-Security-Model]]
