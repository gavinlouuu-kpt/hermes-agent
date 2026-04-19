# 12 · Gateway & Messaging

`hermes gateway start` runs a single long-lived process that listens on
every configured messaging platform and feeds incoming messages into a
per-session `AIAgent`. One process serves Telegram, Discord, Slack,
WhatsApp, Signal, Matrix, Mattermost, DingTalk, Feishu, QQ, Home
Assistant, email, and SMS — with session pairing, delivery routing, and
platform-specific presentation.

## Key files

- [`gateway/run.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/run.py) — `GatewayRunner` orchestrator
- [`gateway/config.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/config.py) — platform config schema
- [`gateway/session.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/session.py) — session store, context prompts
- [`gateway/delivery.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/delivery.py) — message routing
- [`gateway/pairing.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/pairing.py) — multi-user pairing / DM allowlists
- [`gateway/channel_directory.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/channel_directory.py) — user / channel mappings
- [`gateway/stream_consumer.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/stream_consumer.py) — async message stream
- [`gateway/hooks.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/gateway/hooks.py) + [`gateway/builtin_hooks/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway/builtin_hooks)
- [`gateway/platforms/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway/platforms) — adapters

## Supported platforms

| Platform | Adapter | Extra | Notes |
|---|---|---|---|
| Telegram | `platforms/telegram.py` | `messaging` | Long polling or webhook |
| Discord | `platforms/discord_adapter.py` | `messaging` | Slash commands + voice |
| Slack | `platforms/slack.py` | `slack` or `messaging` | Socket Mode |
| WhatsApp | `platforms/whatsapp.py` | `messaging` + Node bridge | Uses `scripts/whatsapp-bridge/` (Baileys) |
| Signal | `platforms/signal.py` | — | Requires `signal-cli` |
| Matrix | `platforms/matrix.py` | `matrix` (Linux) | E2E encryption via mautrix |
| Mattermost | `platforms/mattermost.py` | — | Webhooks |
| DingTalk | `platforms/dingtalk.py` | `dingtalk` | Alibaba enterprise chat |
| Feishu / Lark | `platforms/feishu.py` | `feishu` | ByteDance enterprise chat |
| QQBot | `platforms/qqbot.py` | — | Tencent QQ |
| Email | `platforms/email.py` | — | IMAP poll + SMTP send |
| SMS | `platforms/sms.py` | `sms` | Webhook-based |
| Home Assistant | `platforms/homeassistant.py` | `homeassistant` | Voice assistants, automations |

## Startup sequence

1. `hermes gateway start` → `GatewayRunner` reads `config.yaml` and `.env`.
2. Each enabled platform spawns its adapter as an asyncio task.
3. Adapters authenticate (bot tokens, OAuth, QR pairing for WhatsApp).
4. A health endpoint binds to `:8642` for liveness / readiness checks.
5. Every incoming message hits `_route_user_message()` → session lookup
   → `AIAgent.run_conversation`.
6. Agent responses stream back through the adapter's send path.

## Per-session `AIAgent` cache

The gateway caches `AIAgent` instances per session (one per Telegram user,
one per Slack DM, etc.). The cache is LRU with a 1-hour idle TTL:

- Keep warm for responsive conversations
- Evict to free memory on large deployments
- A cold cache-miss rebuilds the agent from SQLite history

## Session model

A `Session` (see `gateway/session.py`) tracks:

- Platform + user/channel identifier
- Current model + reasoning effort
- Approval context (who approved what, and when)
- Memory/skills overrides (per-platform toolset)
- Delivery preferences (streaming vs. single message, emoji usage)

`gateway/config.py` has platform-level defaults; users override via slash
commands (`/model`, `/personality`, …).

## Pairing & allowlists

Security comes from `gateway/pairing.py` + env allowlists:

- `TELEGRAM_ALLOWED_USERS`, `SLACK_ALLOWED_USERS`, `WHATSAPP_ALLOWED_USERS`,
  `EMAIL_ALLOWED_USERS`, …
- `GATEWAY_ALLOW_ALL_USERS=false` by default — unknown users are
  silently ignored.
- First-time pairing can require a shared secret, a QR scan, or explicit
  admin approval depending on platform.

## Delivery routing

`gateway/delivery.py` handles:

- Message chunking when output exceeds platform limits
- Emoji / reaction support per platform
- Streaming updates (e.g. editing the last message on Telegram vs. posting
  incremental chunks on Slack)
- Voice notes: inbound transcription via `tools/transcription_tools.py`,
  outbound via `tools/tts_tool.py`

## Home channel

Each platform can have a "home channel" — the default destination for
unattributed messages (cron jobs, hooks, insights). Configure via
`TELEGRAM_HOME_CHANNEL`, `EMAIL_HOME_ADDRESS`, etc.

## Hooks

Custom webhook handlers live under `gateway/hooks.py`; builtin examples in
`gateway/builtin_hooks/` (e.g. `boot_md.py` posts a daily bootstrap
report). Hooks run on startup and scheduled triggers; they can post to
any platform through `delivery.py`.

## Web dashboard

A separate FastAPI app (`hermes_cli/web_server.py`) exposes a browser UI
that reads the gateway's health endpoint and SQLite state DB. Typically
run in a second container alongside the gateway.

## WhatsApp bridge

WhatsApp requires a Node.js Baileys bridge running alongside the Python
gateway. `scripts/whatsapp-bridge/` contains the bridge; it speaks
JSON-RPC over stdio to `platforms/whatsapp.py`. Pair the bridge by
scanning a QR code on first launch; state persists under
`~/.hermes/platforms/whatsapp/`.

## Pitfalls

- **Don't run the gateway without an allowlist** unless you really want
  any stranger to talk to your agent.
- **Rate limits.** Telegram, Slack, and Discord rate-limit edits; the
  delivery router respects this but custom hooks should too.
- **Voice transcription is lossy.** For critical messages, the gateway
  falls back to sending the raw audio.
- **WhatsApp** sessions can drop; the Node bridge will reconnect, but new
  QR pairs are needed if you move hosts.

## Source of truth

- [`gateway/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/gateway)
- [`scripts/whatsapp-bridge/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/scripts/whatsapp-bridge)
- [`.env.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.env.example) — platform env vars
- [Messaging Gateway guide](https://hermes-agent.nousresearch.com/docs/user-guide/messaging) (upstream)

## See also

- [[07-CLI-Internals]]
- [[15-Scheduling-Cron]]
- [[17-Security-Model]]
