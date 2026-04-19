# 24 · FAQ

Answers to the questions most likely to come up for someone new to
Hermes. When an answer points into the codebase or config, the link is
included.

## Does it run on Windows?

Not natively. Install [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install)
and run the Linux installer inside it. A PowerShell installer exists
([`scripts/install.ps1`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/scripts/install.ps1))
but is best-effort.

## Does it run on Android?

Yes, via Termux. Use the same curl installer; it detects Termux and
installs a curated `.[termux]` extra that excludes voice dependencies
with Android-incompatible wheels. See
[`constraints-termux.txt`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/constraints-termux.txt).

## Does it work offline?

Core conversation loops require an LLM provider. You can run a local
model (Ollama, LM Studio, vLLM) and set the provider to
`local:<model-name>`. The agent itself doesn't require internet once a
local model is running; individual tools (web search, image generation,
Skills Hub) still do.

## Which model should I use?

- **Best all-round quality**: Anthropic Claude Sonnet 4 / 4.5 via
  OpenRouter or direct API.
- **Open-source**: Nous Portal's Hermes-4-405B, Qwen, Kimi.
- **Local / free**: an Ollama-hosted Llama or Qwen model.
- **Cheap defaults**: GPT-4o-mini via OpenRouter for casual use.

Switch at any time with `/model` — see [[03-Quickstart]].

## How much does it cost to run?

- **LLM inference** is the dominant cost. Depends on the model and
  traffic. Casual use with Claude Sonnet typically runs a few dollars per
  month.
- **Infrastructure**: a $5/mo VPS is plenty for a personal gateway. Modal
  and Daytona idle at near-zero when not in use.
- **Embeddings / search APIs** (Exa, Firecrawl, Parallel): pay-as-you-go.
- Run `/usage` to see per-session cost.

## Can I use it without giving my data to a third party?

Yes:

- Run a local LLM (Ollama, vLLM, LM Studio).
- Set `memory.provider: builtin` (no external memory).
- Avoid external tools (`web_search`, Browserbase, TTS providers).
- Use the `local` terminal backend.

Under that configuration, conversations stay on your machine.

## Can I swap providers mid-conversation?

Yes — `/model provider:model` changes the model for the next turn. Tool
schemas and context carry over. Different providers may disagree about
fine-grained tool-call formats; adapters in `agent/` normalize the
differences.

## How do I reset everything?

```bash
rm -rf ~/.hermes          # nukes all state
hermes setup              # start fresh
```

Or more surgically:

```
/reset                    # new session, same config
```

Profiles help keep personal and work state separate without full resets —
see [[16-Configuration]].

## Where are my conversations stored?

SQLite: `~/.hermes/state.db`. Full-text searchable via `/search`. Export
one with `hermes sessions export <id>`. See [[11-Memory-System]].

## Can I migrate from OpenClaw?

Yes. During `hermes setup`, if `~/.openclaw/` exists the wizard offers
migration. Or later:

```bash
hermes claw migrate              # full interactive migration
hermes claw migrate --dry-run    # preview
hermes claw migrate --preset user-data   # skip secrets
```

## Why is the first request slow after idle?

If your terminal backend is `modal` or `daytona`, the sandbox hibernates
and needs to wake (10–30 seconds). For snappy always-on, use `local`,
`docker`, or `ssh`.

## How do I add my own tool / skill / platform?

See [[21-Extending-Hermes]].

## How do I make the agent run scheduled jobs?

Use the cron system — natural-language jobs dispatched by the gateway.
See [[15-Scheduling-Cron]].

## Can multiple people share one bot?

Yes — the messaging gateway assigns a per-user session (per Telegram
chat, per Slack DM, etc.) and isolates state. Add the user IDs to the
platform's `ALLOWED_USERS` env var. See [[12-Gateway-And-Messaging]].

## How do I prevent dangerous commands?

The approval gate is on by default. Review patterns in
[`tools/approval.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/approval.py)
and customize the allowlist in `~/.hermes/config.yaml` under
`approval.patterns`. See [[17-Security-Model]].

## How do I update?

```bash
hermes update
```

Docker: `docker pull nousresearch/hermes-agent && docker restart hermes`.
Nix: update the flake input.

## Is there a web UI?

Yes — `hermes web` (requires the `web` extra) or the Docker dashboard
container. See [[13-UI-Surfaces]].

## Where's the full documentation?

<https://hermes-agent.nousresearch.com/docs/>. This wiki is a
distillation for in-repo browsing; the upstream Docusaurus site is the
authoritative reference.

## How do I report a bug or security issue?

- **Bugs**: <https://github.com/NousResearch/hermes-agent/issues>
- **Security**: `security@nousresearch.com` (see
  [`SECURITY.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/SECURITY.md))

## What's the license?

MIT. See
[`LICENSE`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/LICENSE).

## Who made it?

[Nous Research](https://nousresearch.com). Join the
[Discord](https://discord.gg/NousResearch).

## See also

- [[Home]]
- [[01-Overview]]
- [[23-Glossary]]
