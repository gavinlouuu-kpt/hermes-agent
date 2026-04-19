# 16 · Configuration

Hermes reads configuration from three sources in order of precedence:
command-line flags → environment variables → `~/.hermes/config.yaml`.
Secrets live in a separate `~/.hermes/.env`. OAuth tokens in
`~/.hermes/auth.json`. Multiple **profiles** under
`~/.hermes/profiles/<name>/` isolate settings per workspace.

## File layout under `~/.hermes/`

```
~/.hermes/
├── config.yaml        # master settings
├── .env               # API keys / platform tokens
├── auth.json          # OAuth credentials
├── SOUL.md            # optional personality
├── state.db           # SQLite + FTS5
├── memories/          # MEMORY.md, USER.md
├── skills/            # user/agent-created skills
├── sessions/          # gateway session state
├── cron/              # scheduled jobs
├── hooks/             # event hooks
├── logs/              # errors.log, gateway.log
├── skins/             # CLI themes
├── plans/             # agent work plans
├── workspace/         # working-directory data
├── home/              # subprocess HOME
├── cache/             # images, audio, browser state
├── platforms/         # per-platform state (e.g. WhatsApp QR pairing)
└── profiles/<name>/   # per-profile overrides
```

`hermes_constants.get_hermes_home()` resolves this path, honouring
`HERMES_HOME` / `XDG_CONFIG_HOME` if set.

## `config.yaml`

The canonical template is
[`cli-config.yaml.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli-config.yaml.example)
at the repo root. Copy it when bootstrapping manually:

```bash
cp cli-config.yaml.example ~/.hermes/config.yaml
```

Key sections (abbreviated):

```yaml
model:
  default: openrouter:anthropic/claude-sonnet-4
  reasoning: medium           # none | minimal | low | medium | high | xhigh

terminal:
  env: local                  # local | docker | ssh | modal | daytona | singularity
  timeout: 120
  lifetime_seconds: 3600
  cwd: ~/workspace
  docker:
    image: nousresearch/hermes-workspace:latest
  ssh:
    host: my-vps.example.com
    user: agent

toolsets:
  default: [terminal, web, memory, skills]
  telegram: [web, memory]
  cron: [web, memory, terminal]

memory:
  provider: builtin           # builtin | honcho | mem0 | retaindb | …
  honcho:
    api_key: ${HONCHO_API_KEY}
    workspace_id: …

network:
  force_ipv4: false

context_compression:
  enabled: true
  threshold: 0.85

approval:
  patterns:
    - "rm -rf"
    - "sudo rm"
  cron_allowlist:
    - "docker system prune -f"

mcp_servers:
  - name: time
    transport: stdio
    command: uvx
    args: [mcp-server-time]

gateway:
  telegram:
    enabled: true
    allowed_users: [123456789]
    home_channel: 123456789
  slack:
    enabled: false

voice:
  tts:
    provider: edge            # edge | elevenlabs | openai | minimax | mistral
  stt:
    provider: faster-whisper  # faster-whisper | openai | groq
```

## `.env` — secrets

The annotated template is
[`.env.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.env.example).
Common keys:

- **LLM providers**: `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`,
  `OPENAI_API_KEY`, `GOOGLE_API_KEY` / `GEMINI_API_KEY`, `OLLAMA_API_KEY`,
  `GLM_API_KEY`, `KIMI_API_KEY`, `ARCEEAI_API_KEY`, `MINIMAX_API_KEY`,
  `HF_TOKEN`, `XIAOMI_API_KEY`, `OPENCODE_ZEN_API_KEY`
- **Tools**: `EXA_API_KEY`, `PARALLEL_API_KEY`, `FIRECRAWL_API_KEY`,
  `FAL_KEY`, `HONCHO_API_KEY`
- **Browser**: `BROWSERBASE_API_KEY`, `BROWSERBASE_PROJECT_ID`,
  `BROWSERBASE_PROXIES`, `BROWSERBASE_ADVANCED_STEALTH`
- **Voice**: `VOICE_TOOLS_OPENAI_KEY`, `GROQ_API_KEY`,
  `STT_GROQ_MODEL`, `STT_OPENAI_MODEL`
- **Terminal backends**: `TERMINAL_ENV`, `TERMINAL_DOCKER_IMAGE`,
  `TERMINAL_SSH_HOST`, `TERMINAL_SSH_USER`, `TERMINAL_SSH_PORT`,
  `TERMINAL_SSH_KEY`, `TERMINAL_MODAL_IMAGE`, `TERMINAL_SINGULARITY_IMAGE`,
  `TERMINAL_CWD`, `TERMINAL_TIMEOUT`, `TERMINAL_LIFETIME_SECONDS`,
  `SUDO_PASSWORD`
- **Messaging**: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`,
  `TELEGRAM_HOME_CHANNEL`, `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`,
  `SLACK_ALLOWED_USERS`, `WHATSAPP_ENABLED`, `WHATSAPP_ALLOWED_USERS`,
  `EMAIL_ADDRESS`, `EMAIL_PASSWORD`, `EMAIL_IMAP_HOST`, `EMAIL_SMTP_HOST`,
  `EMAIL_POLL_INTERVAL`, `EMAIL_ALLOWED_USERS`, `EMAIL_HOME_ADDRESS`,
  `GATEWAY_ALLOW_ALL_USERS`
- **Response pacing**: `HERMES_HUMAN_DELAY_MODE`,
  `HERMES_HUMAN_DELAY_MIN_MS`, `HERMES_HUMAN_DELAY_MAX_MS`
- **RL**: `TINKER_API_KEY`, `WANDB_API_KEY`, `RL_API_URL`
- **GitHub (Skills Hub)**: `GITHUB_TOKEN`, `GITHUB_APP_ID`,
  `GITHUB_APP_PRIVATE_KEY_PATH`, `GITHUB_APP_INSTALLATION_ID`
- **Compression**: `CONTEXT_COMPRESSION_ENABLED`,
  `CONTEXT_COMPRESSION_THRESHOLD`
- **Webhooks**: `TELEGRAM_WEBHOOK_URL`, `TELEGRAM_WEBHOOK_PORT`,
  `TELEGRAM_WEBHOOK_SECRET`

## Profiles

Switch profiles by setting `HERMES_PROFILE` or passing `--profile <name>`:

```bash
HERMES_PROFILE=work hermes
hermes --profile personal
```

Each profile has its own `config.yaml`, `.env`, memories, skills, and
session DB under `~/.hermes/profiles/<name>/`. Ideal for keeping work and
personal state isolated.

## Configuration commands

```bash
hermes config get model.default
hermes config set terminal.env docker
hermes config edit             # opens $EDITOR
```

Implemented in
[`hermes_cli/config.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/config.py)
with atomic writes (`utils.atomic_write_yaml`).

## Provider selection

Providers are identified as `<provider>:<model>`:

- `openrouter:anthropic/claude-sonnet-4`
- `nousportal:Hermes-4-405B`
- `anthropic:claude-sonnet-4-5`
- `openai:gpt-4o`
- `gemini:gemini-2.5-pro`
- `local:llama3.1:70b`

`agent/model_metadata.py` fetches context windows and capability flags
(thinking, vision, tools). `hermes_cli/models.py` holds the browseable
catalog shown by `/model`.

## Env-var expansion in YAML

Values like `${HONCHO_API_KEY}` are expanded at load time from the
environment (which includes `~/.hermes/.env` via `dotenv`). This keeps
secrets out of `config.yaml`.

## Config precedence recap

1. CLI flags (`--model`, `--provider`, `--profile`, `--config-dir`,
   `--toolsets`, `--offline`, …)
2. Environment vars (`HERMES_*`, provider keys, platform tokens)
3. `~/.hermes/profiles/<name>/config.yaml` if profile active
4. `~/.hermes/config.yaml`
5. Built-in defaults in `hermes_cli/config.py`

## Source of truth

- [`cli-config.yaml.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cli-config.yaml.example)
- [`.env.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.env.example)
- [`hermes_cli/config.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/config.py)
- [`hermes_constants.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_constants.py)
- [Configuration guide](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) (upstream)

## See also

- [[02-Installation]]
- [[07-CLI-Internals]]
- [[17-Security-Model]]
- [[18-Deployment]]
