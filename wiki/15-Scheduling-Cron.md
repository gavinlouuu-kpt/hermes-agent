# 15 · Scheduling & Cron

Hermes has a built-in cron scheduler — natural-language jobs that run
unattended and deliver results to any configured messaging platform.
Instead of writing shell scripts, you say "every day at 9am, summarise my
unread GitHub notifications and post to #eng-standup".

## Key files

- [`cron/scheduler.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cron/scheduler.py) — async scheduler loop
- [`cron/jobs.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/cron/jobs.py) — `Job` model + storage
- [`tools/cronjob_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/cronjob_tools.py) — tool for agent to manage jobs
- Delivery + config glue inside `cron/`

Requires the `cron` extra (`croniter`): `uv pip install -e ".[cron]"`
(included in `.[all]` and `.[termux]`).

## The `Job` model

Stored as YAML/JSON under `~/.hermes/cron/`:

```yaml
id: 01HK7A…
name: daily-github-summary
cron: "0 9 * * *"
prompt: |
  Summarise my unread GitHub notifications from the past 24 hours.
  Include which repos they came from and the most urgent one.
target:
  platform: slack
  channel: "#eng-standup"
model: openrouter:anthropic/claude-sonnet-4
tools: [web, terminal, memory]
enabled: true
last_run: 2026-04-18T09:00:00Z
```

Fields:

- `cron` — standard 5-field crontab expression
- `prompt` — natural-language task for the agent
- `target` — where the result is delivered (any configured platform)
- `model`, `tools` — optional overrides for this job
- `enabled` — toggle without deleting

## Managing jobs

### From the CLI

```bash
hermes cron list
hermes cron add "every day at 9am" "summarise my github notifications"
hermes cron disable daily-github-summary
hermes cron run daily-github-summary   # trigger now
hermes cron remove daily-github-summary
```

### From conversation (via the agent)

```
/cron add "every weekday at 8pm" "remind me to log off and read a book"
/cron list
```

The agent uses `tools/cronjob_tools.py` which wraps the same API.

### Natural-language → cron

Phrases like *"every day at 9am"*, *"every weekday at 6pm"*,
*"every hour"*, *"every 15 minutes"*, *"every monday at 9:30"* are parsed
into crontab expressions. For edge cases, pass a raw crontab string
(`"*/10 * * * *"`).

## Scheduler lifecycle

The scheduler runs inside the gateway process (`hermes gateway start`):

1. On startup, `scheduler.py` loads all enabled jobs.
2. Each tick (every minute), `croniter` computes which jobs are due.
3. Due jobs create a new session, spawn an `AIAgent`, and run the prompt.
4. The response is routed through `gateway/delivery.py` to the configured
   target.
5. `last_run` is updated and persisted.

If the gateway is down when a job is due, the job is skipped (no
catch-up) — cron is for recurring reports, not guaranteed queued work.

## Delivery targets

Any messaging platform + channel is valid. Cron delivery uses the same
router as user replies, so rich output (Markdown, code blocks, images,
voice) is rendered per platform.

A special `home` target uses the platform's configured home channel
(`TELEGRAM_HOME_CHANNEL`, `EMAIL_HOME_ADDRESS`, …) — convenient when you
want one inbox for all scheduled reports.

## Per-job config overrides

Jobs can override the default model, reasoning effort, and toolset:

```yaml
model: nousportal:Hermes-4-405B
reasoning: low
tools: [web, memory]
```

This lets you run expensive reasoning for daily reports and a cheap local
model for hourly health-checks.

## Approval gates and cron

Cron jobs run **without a user in the loop**. They cannot prompt for
approval, so the scheduler blocks any tool call that would trigger the
approval gate. Use a pre-approved allowlist in config if you need a cron
job to run dangerous operations:

```yaml
approval:
  cron_allowlist:
    - "docker system prune -f"
```

## Pitfalls

- **Gateway must be running.** Jobs only fire while `hermes gateway start`
  is up. On a VPS, keep it running under systemd/docker/tmux.
- **No overlap protection.** A hourly job that sometimes takes 90 minutes
  will overlap with itself. Keep jobs fast or add idempotency checks.
- **Timezones.** Cron expressions interpret in the gateway process's
  timezone (`TZ` env). Pin `TZ=UTC` if you deploy across zones.
- **Catch-up isn't performed.** Missed runs are missed. Don't use cron
  for data pipelines where every run must happen.

## Source of truth

- [`cron/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/cron)
- [`tools/cronjob_tools.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/cronjob_tools.py)
- [Cron guide](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) (upstream)

## See also

- [[12-Gateway-And-Messaging]]
- [[17-Security-Model]]
- [[18-Deployment]]
