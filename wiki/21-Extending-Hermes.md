# 21 · Extending Hermes

Four common extension points, from easiest to most involved:

1. Write a **skill** (Markdown + optional scripts)
2. Add a **tool** (Python module with a registered handler)
3. Add a **memory provider** or **MCP server** (plugin)
4. Add a **messaging platform** (gateway adapter)

Each below includes the minimum viable implementation; study the existing
code in the referenced directories for fuller examples.

## 1 · Adding a skill

Fastest path: `hermes skills new <name>` scaffolds a directory under
`~/.hermes/skills/`.

Minimum `SKILL.md`:

```markdown
---
name: productivity:shopping-list
description: Track a persistent shopping list in ~/shopping.md.
triggers:
  - "add to shopping list"
  - "what's on my shopping list"
toolsets_required: [terminal]
---

# Instructions

- When the user asks to add items: append lines to `~/shopping.md`.
- When the user asks to view: read `~/shopping.md` and render as a list.
- When the user asks to clear: prompt for confirmation, then `rm ~/shopping.md`.
```

Test with `/productivity:shopping-list add eggs and bread`.

Publish with `hermes skills publish .` (requires a Skills Hub account; the
guard in `tools/skills_guard.py` scans your skill before upload).

See [[10-Skills-System]] for the full authoring model.

## 2 · Adding a tool

Create `tools/my_tool.py`:

```python
from .registry import registry

SCHEMA = {
    "type": "function",
    "function": {
        "name": "weather",
        "description": "Get the current weather for a city.",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "City name."}
            },
            "required": ["city"],
        },
    },
}

async def handle_weather(args, ctx):
    city = args["city"]
    # ctx has the current session, config, approval callback, etc.
    result = await ctx.http.get(f"https://wttr.in/{city}?format=3")
    return {"city": city, "report": result.text}

registry.register(
    name="weather",
    schema=SCHEMA,
    handler=handle_weather,
    toolset="productivity",
    is_available=lambda cfg: True,
)
```

Then:

- Add `weather` to the default toolset in `toolsets.py` (or rely on user
  config).
- If it's destructive, add a pattern in `tools/approval.py`.
- Write a pytest case under `tests/tools/test_weather.py`.
- Restart Hermes — the tool auto-registers on import.

Reference patterns:

- [`tools/web_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/web_tools.py) for HTTP tools
- [`tools/homeassistant_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/homeassistant_tool.py) for config-driven tools
- [`tools/terminal_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/terminal_tool.py) for multi-backend tools

## 3 · Adding a memory provider

Memory providers implement a common interface (prefetch / sync / search).
They live under [`plugins/memory/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/plugins).

Minimum skeleton:

```python
# plugins/memory/my_provider/provider.py
from agent.memory_provider import MemoryProvider

class MyProvider(MemoryProvider):
    def __init__(self, config):
        self.endpoint = config["endpoint"]
        self.api_key = config["api_key"]

    async def prefetch(self, session_id, user_message):
        return await self._call("prefetch", {"q": user_message})

    async def sync(self, session_id, user_msg, assistant_msg):
        await self._call("sync", {"u": user_msg, "a": assistant_msg})

    async def search(self, query, limit=10):
        return await self._call("search", {"q": query, "n": limit})
```

Register it under `plugins/memory/__init__.py` so `memory_manager` picks
it up, then enable via `config.yaml`:

```yaml
memory:
  provider: my_provider
  my_provider:
    endpoint: https://api.example.com
    api_key: ${MY_PROVIDER_KEY}
```

Reference implementations: `honcho/`, `mem0/`, `retaindb/`.

## 4 · Adding an MCP server

You can make a custom capability available to Hermes without adding a
Hermes-specific tool: write an MCP server in any language and add it to
`mcp_servers:` in `config.yaml`.

The `fastmcp` optional skill
([`optional-skills/mcp/fastmcp/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/optional-skills/mcp/fastmcp))
provides a small Python framework for MCP servers.

```python
# my_server.py
from fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool()
def current(city: str) -> str:
    return fetch_weather(city)

if __name__ == "__main__":
    mcp.run_stdio()
```

Then in `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  - name: weather
    transport: stdio
    command: python
    args: [my_server.py]
```

See [[14-MCP-Integration]] for transport options and security.

## 5 · Adding a messaging platform

This is the most involved extension. Adapters live under
[`gateway/platforms/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway/platforms).

A platform adapter must:

1. Inherit from the base `PlatformAdapter` in `gateway/platforms/base.py`.
2. Implement authentication (bot token, OAuth, QR pair).
3. Implement `receive` — push inbound messages into the gateway's queue.
4. Implement `send` — deliver outbound text / media / reactions.
5. Implement `allowlist_check`.
6. Register with `GatewayRunner` in `gateway/run.py`.

Smallest reference: `gateway/platforms/sms.py` (webhook-based). Add a new
extra in `pyproject.toml` for any platform-specific deps (`mautrix`,
`slack-sdk`, etc.).

Once added, the same slash commands and approval flow apply
automatically — you don't need to reimplement those per platform.

## 6 · Plugins: `plugins/context_engine/`

For cross-cutting features that don't fit a tool or memory provider, the
`plugins/context_engine/` system lets you inject processed context into
the system prompt. Used e.g. to add build-time project summaries.

## Testing your extension

Place tests under `tests/<area>/`. The suite uses `pytest-asyncio` and
`pytest-xdist`. Example:

```bash
pytest tests/tools/test_my_tool.py -xvs
```

Fixtures for fake LLMs, fake registries, and fake sessions live under
`tests/fakes/` and `tests/conftest.py`.

## Publishing checklist

- [ ] Add unit tests
- [ ] Run `pytest tests/ -q` locally
- [ ] Update `CONTRIBUTING.md` or `AGENTS.md` if the extension changes a
      stable interface
- [ ] Add a release note snippet for the next `RELEASE_v*.md`
- [ ] Open a PR — CI will run `supply-chain-audit.yml`,
      `contributor-check.yml`, and `tests.yml`

## Source of truth

- [`tools/registry.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/registry.py) — tool registration
- [`agent/memory_provider.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/memory_provider.py) — memory plugin base
- [`gateway/platforms/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway/platforms) — platform adapters
- [`plugins/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/plugins) — plugins
- [`CONTRIBUTING.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/CONTRIBUTING.md)

## See also

- [[08-Tool-System]]
- [[10-Skills-System]]
- [[11-Memory-System]]
- [[12-Gateway-And-Messaging]]
- [[14-MCP-Integration]]
- [[22-Contributing]]
