# 09 · Terminal Backends

The `terminal` tool is the agent's hands. It runs shell commands through
one of six interchangeable backends: local subprocess, Docker container,
SSH session, Modal function, Daytona workspace, or Singularity/Apptainer
container. All six implement the same `EnvironmentBase` interface so the
agent code is backend-agnostic.

## Where they live

- Dispatch & high-level tool: [`tools/terminal_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/terminal_tool.py)
- Per-backend drivers:
  - [`tools/environments/base.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/base.py)
  - [`tools/environments/local.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/local.py)
  - [`tools/environments/docker.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/docker.py)
  - [`tools/environments/ssh.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/ssh.py)
  - [`tools/environments/modal.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/modal.py)
  - [`tools/environments/daytona.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/daytona.py)
  - [`tools/environments/singularity.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/environments/singularity.py)

## Backend comparison

| Backend | Persistent? | Isolation | Best for | Requires |
|---|---|---|---|---|
| `local` | Yes (the host shell) | None | Personal use on a trusted host | Nothing |
| `docker` | Per-container | Container | Safer local sandbox | `docker` or `podman` in `$PATH` |
| `ssh` | Yes (remote host) | Remote user | Agent on VPS, editing on laptop | SSH key / agent |
| `modal` | Hibernates when idle | Modal function | Serverless; nearly-free idle | `modal` extra + Modal account |
| `daytona` | Hibernates when idle | Daytona workspace | Serverless dev envs | `daytona` extra + Daytona account |
| `singularity` | Per-container | HPC container | Shared clusters, scientific computing | Singularity/Apptainer |

## Selecting a backend

Set in `~/.hermes/config.yaml`:

```yaml
terminal:
  env: docker              # local | docker | ssh | modal | daytona | singularity
  timeout: 120             # seconds
  lifetime_seconds: 3600   # keep session warm this long
  cwd: /workspace
  docker:
    image: nousresearch/hermes-workspace:latest
    volumes:
      - /data:/data
  ssh:
    host: my-vps.example.com
    user: agent
    port: 22
    key: ~/.ssh/id_ed25519
```

Environment-variable equivalents (see [`.env.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.env.example)):
`TERMINAL_ENV`, `TERMINAL_DOCKER_IMAGE`, `TERMINAL_CWD`, `TERMINAL_TIMEOUT`,
`TERMINAL_LIFETIME_SECONDS`, `TERMINAL_SSH_HOST`, …

## Per-backend notes

### `local`
Spawns subprocesses under the current user via `ptyprocess` (Linux/macOS)
or `pywinpty` (Windows). No isolation — everything the agent runs has your
UID. Good for single-user, trusted scenarios.

### `docker`
Runs commands inside a long-lived container (image configurable). Supports
volume mounts for persistent state and GPU passthrough. Use this on a
shared server or when you want the agent sandboxed from the host. Override
the binary with `HERMES_DOCKER_BINARY=podman` if you prefer Podman.

### `ssh`
Maintains one pooled SSH session (via OpenSSH control-master) and runs
commands on the remote host. Great for "agent lives on VPS, I edit on my
laptop" setups. Supports key-based or agent-forwarded auth.

### `modal`
Each session opens a Modal function; commands run inside a cloud sandbox
that **hibernates** when idle and wakes on demand. Cost is near-zero
between sessions. Requires the `modal` extra and a Modal account.

### `daytona`
Similar to Modal but uses Daytona workspaces — ephemeral cloud dev
environments with filesystem persistence across suspend/resume.

### `singularity`
For HPC clusters where Docker is unavailable but Singularity/Apptainer is.

## The `terminal` tool contract

```
terminal(command, cwd=None, timeout=None, background=False)
→ {
    stdout: str,
    stderr: str,
    exit_code: int,
    duration_ms: int,
  }
```

For `background=True`, the tool registers a process via
`tools/process_registry.py`, returns a handle, and exposes `poll` / `kill`
tools for the agent to manage it.

## Approval integration

Every backend runs its command through the same `approval` gate. A
destructive pattern in `tools/approval.py` (e.g. `rm -rf /`,
`dd of=/dev/...`, `sudo` without an allowlisted target) fires the approval
callback regardless of which backend would execute it.

## Persistent shells

Backends can cache a shell session for `lifetime_seconds`, so `export FOO=…`
in one turn persists into the next. The CWD is tracked via
`agent/subdirectory_hints.py`, which injects the current working directory
into the system prompt so the model knows where it is.

## Pitfalls

- **Don't mix backends mid-session.** Switching `terminal.env` after the
  agent has built up state resets the shell.
- **Modal/Daytona cold start.** First command may take 10–30 seconds.
- **`ssh` requires agent forwarding if you want the remote shell to reach
  git etc.** Set `ForwardAgent yes` in `~/.ssh/config` for the target.
- **Sudo passwords.** Set `SUDO_PASSWORD` in `~/.hermes/.env` or accept
  the approval prompt each time.

## Source of truth

- [`tools/terminal_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/terminal_tool.py)
- [`tools/environments/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tools/environments)
- [`.env.example`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.env.example)

## See also

- [[08-Tool-System]]
- [[17-Security-Model]]
- [[18-Deployment]]
