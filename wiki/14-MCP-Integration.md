# 14 · MCP Integration

Hermes speaks the **Model Context Protocol** (MCP) in both directions: it
is an **MCP client** — pulling tools and resources from any MCP server
into the agent loop — and an **MCP server** — exposing Hermes's own tools
to other MCP-aware clients.

## Why MCP

MCP standardizes how agents discover and call external tools. Instead of
writing a custom adapter per API, you point Hermes at an MCP server and
its tools join the registry at runtime. Examples: filesystem, GitHub,
Notion, a time service, a Slack workspace, a private corporate API.

## Client: loading MCP servers into Hermes

Config lives under `mcp_servers:` in `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  - name: time
    transport: stdio
    command: uvx
    args: [mcp-server-time]
  - name: filesystem
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/docs"]
  - name: notion
    transport: http
    url: https://mcp.notion.com/
    auth:
      header: Authorization
      value: ${NOTION_MCP_TOKEN}
  - name: github
    transport: stdio
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: ${GITHUB_TOKEN}
```

The client implementation is
[`tools/mcp_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/mcp_tool.py).
At startup it:

1. Launches each stdio server as a subprocess or opens an HTTP session.
2. Negotiates the MCP handshake.
3. Enumerates the server's tools and registers them in the Hermes tool
   registry with a namespaced prefix (`mcp:<name>:<tool>`).
4. Exposes MCP resources through the session-search interface.
5. Optionally enables **sampling** — letting the MCP server request LLM
   completions back through Hermes (off by default per server).

## Server: exposing Hermes as MCP

[`mcp_serve.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/mcp_serve.py)
runs Hermes's tool registry as a standalone MCP server. Other clients
(Claude Desktop, Zed, IDE plugins, other agents) can connect and use the
exact toolset your Hermes install has configured.

```bash
python mcp_serve.py --transport stdio
# or HTTP mode on a specified port
python mcp_serve.py --transport http --port 7611
```

This is useful when you want another agent (or an editor) to reuse
Hermes's terminal, browser, memory, and skills without reimplementing
them.

## Bundled MCP-related skills

- [`skills/mcp/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/skills/mcp) —
  `mcporter` skill wraps arbitrary MCP tools for invocation shortcuts.
- [`optional-skills/mcp/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/optional-skills/mcp) —
  includes `fastmcp` (MCP server framework) for rapid custom-server
  development.

## Security considerations

Every MCP server is a remote code path that can influence what the agent
does. Treat them like skills:

- Only add MCP servers from trusted sources.
- Review the server's `tools/list` output before giving it approval-free
  access.
- Sampling (server-initiated LLM calls) is off by default; enable per-server
  only when needed and budget for the extra tokens.
- For HTTP servers, pin to a specific URL and use auth headers rather
  than anonymous access.

## Install extra

Requires the `mcp` extra:

```bash
uv pip install -e ".[mcp]"
# or .[all] which includes it
```

Under the hood this installs the official `mcp` Python SDK (version
pinned in `pyproject.toml`).

## Pitfalls

- **stdio servers can print to stderr.** Noise floods the log; redirect or
  silence on servers you don't debug.
- **Long-running MCP tools.** The client awaits the server's response; a
  slow MCP call blocks the turn. Use `timeout` on the MCP server where
  supported.
- **Tool name collisions.** If two MCP servers both expose `search`, the
  namespaced prefix (`mcp:notion:search`) prevents collision but you may
  need aliasing in skills.

## Source of truth

- [`tools/mcp_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/mcp_tool.py) — client
- [`mcp_serve.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/mcp_serve.py) — server
- [`skills/mcp/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/skills/mcp), [`optional-skills/mcp/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/optional-skills/mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)

## See also

- [[08-Tool-System]]
- [[10-Skills-System]]
- [[21-Extending-Hermes]]
