# Hermes Agent Wiki

**Hermes Agent** is Nous Research's self-improving AI assistant: an agent with
a built-in learning loop that creates skills from experience, searches its own
past conversations, and builds a deepening model of who you are across
sessions. It runs on your laptop, a $5 VPS, or serverless infrastructure; you
can talk to it from the terminal, Telegram, Discord, Slack, WhatsApp, Signal,
email, and more — all from one gateway process.

This wiki distills the repository into a learnable shape: what's here, how the
pieces fit together, and how to extend them.

## Start here

| If you are… | Read |
|---|---|
| New to Hermes | [[01-Overview]] → [[02-Installation]] → [[03-Quickstart]] |
| Reading the code | [[04-Repository-Layout]] → [[05-Architecture]] → [[06-Agent-Loop]] |
| Configuring a deployment | [[16-Configuration]] → [[18-Deployment]] → [[17-Security-Model]] |
| Writing a tool / skill / platform | [[21-Extending-Hermes]] → [[08-Tool-System]] |
| Working on RL training | [[20-RL-And-Trajectories]] |
| Contributing | [[22-Contributing]] |
| Looking up a term | [[23-Glossary]] · [[24-FAQ]] |

## What Hermes gives you

- **A real terminal interface** — `hermes` drops you into a `prompt_toolkit`
  REPL with multiline editing, slash-command autocomplete, streaming tool
  output, and interrupt-and-redirect.
- **Messaging gateway** — one `hermes gateway` process serves Telegram,
  Discord, Slack, WhatsApp, Signal, email, Matrix, DingTalk, Feishu, QQ,
  Home Assistant, and SMS.
- **A closed learning loop** — per-session memory (`MEMORY.md`, `USER.md`),
  agent-curated skills under `~/.hermes/skills/`, FTS5 search over every
  past conversation, optional Honcho dialectic user modeling.
- **Scheduled automations** — a cron scheduler runs natural-language jobs and
  delivers results to any platform.
- **Runs anywhere** — six terminal backends (local, Docker, SSH, Modal,
  Daytona, Singularity); Modal and Daytona offer serverless persistence so
  your agent environment hibernates when idle.
- **Any model, any provider** — OpenRouter (200+ models), Nous Portal,
  Anthropic, OpenAI, Google Gemini, NVIDIA NIM, Hugging Face, Ollama, or any
  OpenAI-compatible endpoint. Switch with `/model`.

## Project at a glance

- **Language:** Python 3.11+ (Ink/React for the experimental TUI;
  Node.js for the browser and WhatsApp bridges).
- **License:** MIT.
- **Latest release:** v0.10.0 (April 16, 2026) — see [`RELEASE_v0.10.0.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/RELEASE_v0.10.0.md).
- **Core entry points** ([`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml)):
  - `hermes = hermes_cli.main:main`
  - `hermes-agent = run_agent:main`
  - `hermes-acp = acp_adapter.entry:main`
- **State:** SQLite + FTS5 at `~/.hermes/state.db`; config at
  `~/.hermes/config.yaml`; secrets at `~/.hermes/.env`.
- **Tests:** ~3000 in `tests/` (pytest + pytest-asyncio + pytest-xdist).

## A one-paragraph mental model

Every conversation goes through a single class, `AIAgent`, defined at
[`run_agent.py:588`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/run_agent.py#L588).
Its `run_conversation` method (line 8644) loops: build a system prompt →
call the LLM → execute any tool calls via `tools/registry.py` → repeat until
the model returns a final answer. Everything else — the CLI, the messaging
gateway, the cron scheduler, the web dashboard, the RL training environments —
is a different way to feed messages into that same loop.

## Canonical docs

This wiki is a distillation, not a replacement, for the official docs at
**[hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/)**.
Where upstream is more complete (user guides, release notes, API reference),
wiki pages link out.

---

See [[_Sidebar]] for full navigation.
