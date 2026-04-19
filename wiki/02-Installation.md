# 02 · Installation

Hermes runs on Linux, macOS, WSL2, and Android (Termux). Native Windows is
not supported. Three installation paths are supported: a one-line installer
for users, a `setup-hermes.sh` script for contributors, and Docker / Nix for
reproducible deployments.

## Supported platforms

| Platform | Status | Notes |
|---|---|---|
| Linux (x86_64, aarch64) | First-class | Tested in CI |
| macOS (Intel, Apple Silicon) | First-class | Tested in CI (Nix jobs) |
| WSL2 on Windows | First-class | Use this instead of native Windows |
| Android (Termux) | Supported, limited | Use `.[termux]` extra; voice deps are excluded |
| Native Windows | Not supported | [`scripts/install.ps1`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/scripts/install.ps1) exists but is best-effort |

## Path 1 — One-line installer (users)

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc   # or ~/.zshrc
hermes setup       # interactive wizard — pick a model, paste API keys
hermes             # start chatting
```

`scripts/install.sh` detects your platform, clones the repo into
`~/.hermes/src`, runs `setup-hermes.sh`, and symlinks the `hermes` command
into `~/.local/bin`.

## Path 2 — Contributor setup

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh     # installs uv, creates venv, installs .[all], symlinks ~/.local/bin/hermes
./hermes              # auto-detects the venv — no manual `source` needed
```

Manual equivalent:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
python -m pytest tests/ -q
```

### What `setup-hermes.sh` does

- Detects desktop/server vs Termux environment.
- Installs `uv` (or uses system Python on Termux).
- Creates a Python 3.11 virtualenv.
- Picks the install extra:
  `.[all]` on desktops/servers, `.[termux]` on Termux, base install as
  fallback.
- Optionally initialises the `tinker-atropos` submodule for RL training.
- Symlinks `hermes` → `~/.local/bin/hermes` (or `$PREFIX/bin/hermes` on Termux).
- Appends `~/.local/bin` to `$PATH` in `~/.bashrc` / `~/.zshrc`.
- Syncs bundled skills using `tools/skills_sync.py`.
- Offers to run the setup wizard at the end.

## Path 3 — Docker

The [`Dockerfile`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/Dockerfile)
produces a multi-arch image (`amd64`, `arm64`) published to Docker Hub as
`nousresearch/hermes-agent`. It runs as a non-root user (`hermes`, UID
10000) and expects persistent data mounted at `/opt/data`.

```bash
# Interactive chat
mkdir -p ~/.hermes
docker run -it --rm -v ~/.hermes:/opt/data nousresearch/hermes-agent

# Initial setup
docker run -it --rm -v ~/.hermes:/opt/data nousresearch/hermes-agent setup

# Gateway daemon
docker run -d --name hermes --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  nousresearch/hermes-agent gateway run

# Web dashboard (separate container)
docker run -d --name hermes-dashboard --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 9119:9119 \
  -e GATEWAY_HEALTH_URL=http://gateway:8642 \
  nousresearch/hermes-agent dashboard
```

The container entrypoint is
[`docker/entrypoint.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/docker/entrypoint.sh)
— it handles privilege dropping with `gosu`, bootstraps
`~/.hermes/{.env,config.yaml,SOUL.md}` from templates, and syncs bundled
skills on first run.

## Path 4 — Nix / NixOS

```bash
# Run directly
nix run github:NousResearch/hermes-agent

# Development shell
nix develop

# As a NixOS module
# Add the flake to your inputs and include its nixosModule in configuration.nix
```

The flake ([`flake.nix`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/flake.nix))
supports `x86_64-linux`, `aarch64-linux`, and `aarch64-darwin`. Package and
module definitions live under
[`nix/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/nix).

## Path 5 — Android (Termux)

```bash
# Inside Termux:
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Termux uses a curated `.[termux]` extra because the default `.[all]` extra
pulls Android-incompatible voice wheels (`ctranslate2`, `onnxruntime` via
`faster-whisper`). See
[`constraints-termux.txt`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/constraints-termux.txt)
for the excluded packages and the
[Termux guide](https://hermes-agent.nousresearch.com/docs/getting-started/termux)
for the tested manual path.

## Optional install extras

Defined in [`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml)
`[project.optional-dependencies]`:

| Extra | What it adds |
|---|---|
| `messaging` | Telegram, Discord, Slack, aiohttp, QR code generation |
| `matrix` | mautrix with E2E encryption (Linux only) |
| `slack` | Slack-only subset |
| `dingtalk` / `feishu` | Chinese messaging platforms |
| `homeassistant` / `sms` | Smart-home and SMS gateway |
| `voice` | Local STT (`faster-whisper`) + audio I/O |
| `tts-premium` | ElevenLabs TTS |
| `mcp` | Model Context Protocol client/server |
| `honcho` | Honcho dialectic user modeling |
| `modal` / `daytona` | Serverless terminal backends |
| `bedrock` | AWS Bedrock adapter |
| `mistral` | Mistral API adapter |
| `web` | FastAPI dashboard backend |
| `acp` | Agent Client Protocol (Zed / VS Code / JetBrains) |
| `rl` | Atropos RL environment + Tinker + wandb |
| `cron` | Scheduled jobs (`croniter`) |
| `cli` | `simple-term-menu` for richer TUIs |
| `pty` | `ptyprocess` / `pywinpty` terminal backends |
| `dev` | pytest, pytest-asyncio, pytest-xdist, debugpy |
| `all` | Everything above (platform-gated where needed) |
| `termux` | Curated subset safe for Android |

Install multiple extras: `uv pip install -e ".[voice,messaging,dev]"`.

## Post-install layout

After `hermes setup` creates `~/.hermes/`:

```
~/.hermes/
├── config.yaml         # master settings
├── .env                # API keys & secrets
├── auth.json           # OAuth credentials (Nous Portal, etc.)
├── SOUL.md             # optional: agent personality
├── state.db            # SQLite + FTS5: sessions, messages
├── memories/           # MEMORY.md, USER.md
├── skills/             # user- and agent-created skills
├── sessions/           # per-gateway-session conversation state
├── cron/               # scheduled job definitions
├── hooks/              # event hooks
├── logs/               # errors.log, gateway.log
├── skins/              # CLI themes
├── plans/              # agent work plans
├── workspace/          # working-directory data
├── home/               # subprocess HOME for git/ssh/npm
├── cache/              # images, audio, browser state
└── platforms/          # platform-specific state (WhatsApp)
```

## Verifying the install

```bash
hermes doctor   # diagnostics: Python version, API keys, tools, terminals, skills
hermes --help   # subcommand list
hermes model    # interactive model picker (requires a provider key)
```

If `hermes doctor` reports issues, [[24-FAQ]] and
[`hermes_cli/doctor.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/doctor.py)
are good next reads.

## Source of truth

- [`scripts/install.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/scripts/install.sh)
- [`setup-hermes.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/setup-hermes.sh)
- [`Dockerfile`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/Dockerfile)
- [`docker/entrypoint.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/docker/entrypoint.sh)
- [`flake.nix`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/flake.nix)
- [`constraints-termux.txt`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/constraints-termux.txt)
- [`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml) — `[project.optional-dependencies]`

## See also

- [[03-Quickstart]]
- [[16-Configuration]]
- [[18-Deployment]]
- [[24-FAQ]]
