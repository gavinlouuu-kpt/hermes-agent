# 18 · Deployment

How to get Hermes running somewhere persistent: laptop, VPS, container,
Nix box, or serverless cloud. Installation paths are covered in
[[02-Installation]]; this page is about keeping it alive after installation.

## Recommended topologies

### Personal laptop
`hermes` directly in a terminal. Nothing scheduled runs when the lid is
closed, so use this only for interactive work. Ctrl+C interrupts; closing
the terminal ends the session (history persists in `state.db`).

### Cheap VPS (recommended)
A $5/mo VPS running `hermes gateway start` under systemd gives you a
persistent agent reachable from Telegram/Slack/etc. with cron jobs, 24/7.
Typical deployment:

```
systemd unit
 └── hermes gateway start
      ├── all configured platform adapters
      ├── cron scheduler
      └── per-session AIAgent instances (LRU cache)
```

### Docker Compose (self-hosted)
Two containers: `gateway` + `dashboard`, sharing a volume at `/opt/data`.
Dashboard reads the gateway's health endpoint on port 8642 and the
shared SQLite DB.

### Serverless (Modal / Daytona)
Gateway on a VPS, terminal backend set to `modal` or `daytona`. The
execution environment hibernates when idle and wakes on demand — you pay
essentially nothing between sessions.

## systemd unit (bare-metal / VPS)

```ini
# /etc/systemd/system/hermes.service
[Unit]
Description=Hermes Agent Gateway
After=network.target

[Service]
Type=simple
User=hermes
Group=hermes
Environment=HERMES_HOME=/home/hermes/.hermes
Environment=TZ=UTC
ExecStart=/home/hermes/.local/bin/hermes gateway start
Restart=on-failure
RestartSec=5s
# Ensure voice / whisper models cache somewhere writable
Environment=XDG_CACHE_HOME=/home/hermes/.cache

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hermes.service
journalctl -u hermes.service -f
```

## Docker

Image: `nousresearch/hermes-agent` (multi-arch: amd64, arm64).

Entrypoint: [`docker/entrypoint.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/docker/entrypoint.sh) —
drops privileges to UID 10000 via `gosu`, bootstraps
`/opt/data/{.env,config.yaml,SOUL.md}`, syncs bundled skills.

```bash
# Interactive setup
docker run -it --rm -v ~/.hermes:/opt/data nousresearch/hermes-agent setup

# Gateway (persistent)
docker run -d --name hermes --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  nousresearch/hermes-agent gateway start

# Dashboard (optional)
docker run -d --name hermes-dashboard --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 9119:9119 \
  -e GATEWAY_HEALTH_URL=http://hermes:8642 \
  nousresearch/hermes-agent dashboard
```

Example `compose.yaml`:

```yaml
services:
  gateway:
    image: nousresearch/hermes-agent
    restart: unless-stopped
    command: ["gateway", "start"]
    volumes: [ "./data:/opt/data" ]
    ports: [ "8642:8642" ]
    environment:
      TZ: UTC
  dashboard:
    image: nousresearch/hermes-agent
    restart: unless-stopped
    command: ["dashboard"]
    depends_on: [ gateway ]
    volumes: [ "./data:/opt/data" ]
    ports: [ "9119:9119" ]
    environment:
      GATEWAY_HEALTH_URL: http://gateway:8642
```

## NixOS module

`flake.nix` exports a `nixosModule`. Example:

```nix
# configuration.nix
{
  inputs.hermes-agent.url = "github:NousResearch/hermes-agent";
  imports = [ inputs.hermes-agent.nixosModules.default ];
  services.hermes-agent = {
    enable = true;
    user = "hermes";
    group = "hermes";
    dataDir = "/var/lib/hermes";
  };
}
```

Declarative config lives in the module options; secrets should be
populated at `${dataDir}/.env` via deploy tooling (agenix, sops-nix, etc.).

## Serverless terminal backends

Modal and Daytona are configured in `config.yaml`; the gateway itself
still needs a host. The pattern: cheap always-on VPS runs the gateway
(tens of MB of RAM), heavy execution happens in the serverless backend
on demand.

- **Modal**: hibernates after ~60s of idleness, wakes in seconds. Per-call
  billing.
- **Daytona**: workspace-oriented; great when the agent needs filesystem
  continuity between runs.

## Observability

- **Health**: `curl http://localhost:8642/health` returns
  `{"status":"ok","platforms":[...]}`.
- **Logs**: `~/.hermes/logs/errors.log` + `gateway.log` (redacted).
- **Metrics**: token usage + cost persisted to `state.db`; the dashboard
  surfaces weekly/monthly charts.
- **Session export**: `hermes sessions export <id>` writes JSON to stdout.

## Backups

Three things worth backing up:

1. `~/.hermes/state.db` — every conversation ever
2. `~/.hermes/memories/` — `MEMORY.md`, `USER.md`
3. `~/.hermes/skills/` — custom and agent-created skills
4. (Optional) `~/.hermes/config.yaml` + `~/.hermes/.env`

Tools: plain `rsync`, `restic`, `borgbackup`, etc. `hermes_cli/backup.py`
has a helper for export.

## Updating

```bash
hermes update
```

Detects install method (pip, uv, git, Nix, Homebrew) and does the right
thing. On Docker, pull the new image and restart. On Nix, update the
flake input.

## Uninstall

```bash
hermes update --uninstall          # removes symlinks
rm -rf ~/.hermes                   # removes all data
```

Docker: `docker rm -f hermes hermes-dashboard && rm -rf ./data`.

## Pitfalls

- **Forgetting `TZ`.** Cron jobs run in the process timezone. Pin to UTC
  unless you know what you want.
- **Volume ownership in Docker.** The image runs as UID 10000; ensure the
  host volume is writable by that UID (`chown -R 10000:10000 ./data`).
- **Gateway on phone tethering.** Long-poll platforms (Telegram without
  webhooks, Slack Socket Mode) burn battery. Use webhooks for Telegram if
  you can route HTTPS.
- **Rotating API keys.** Update `~/.hermes/.env` and restart the
  gateway — env is read at process start, not per turn.

## Source of truth

- [`Dockerfile`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/Dockerfile)
- [`docker/entrypoint.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/docker/entrypoint.sh)
- [`flake.nix`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/flake.nix) + [`nix/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/nix)
- [`setup-hermes.sh`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/setup-hermes.sh)
- [`packaging/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/packaging) — Homebrew / Debian

## See also

- [[02-Installation]]
- [[09-Terminal-Backends]]
- [[12-Gateway-And-Messaging]]
- [[15-Scheduling-Cron]]
