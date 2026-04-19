# 17 · Security Model

Hermes is a general-purpose agent with tool access — it can run shell
commands, edit files, call APIs, and post to messaging platforms. Its
security model is built on four principles: **approval gating**,
**secrets redaction**, **execution sandboxing**, and **supply-chain
auditing**.

Canonical policy:
[`SECURITY.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/SECURITY.md).

## 1 · Approval gating

Every tool call passes through `tools/approval.py`. Arguments are matched
against pattern sets:

- **Dangerous patterns** (block or prompt): `rm -rf /`, `dd of=/dev/…`,
  piped `curl | bash`, writes to `.ssh/authorized_keys`, edits of
  protected paths, sudo without allowlisted target
- **Network unusual**: requests to unexpected hosts (not in the
  allowlist), DNS exfil heuristics
- **Secret-adjacent**: writes that touch `.env`, `credentials`, `keys/`,
  `id_rsa`, etc.

On a match, the approval callback fires:

- CLI — renders a prompt with the command and asks y/n
- Gateway — sends a DM to the session owner
- Cron — blocks; cron jobs cannot prompt (see
  `approval.cron_allowlist`)
- ACP — the editor renders an approval dialog

Users can **permanently allow** patterns in `~/.hermes/config.yaml`:

```yaml
approval:
  patterns:
    - "rm -rf /tmp/build-*"
    - "docker system prune -f"
```

## 2 · Secrets redaction

[`agent/redact.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/redact.py)
scrubs secrets from:

- Tool output before the LLM sees it
- Log files (`~/.hermes/logs/`)
- Diagnostics from `hermes doctor`
- Trajectory recordings (optional RL data)

Patterns include API keys, bearer tokens, JWTs, private keys, AWS
credentials, `.env`-style `KEY=value` lines. New secret-shaped strings
are classified heuristically (high entropy + known prefix).

The system prompt also carries a rule: *never* quote secrets back into
user-visible output. Models occasionally slip; redaction is defence in
depth.

## 3 · Execution sandboxing

Terminal backends (see [[09-Terminal-Backends]]) provide varying levels
of isolation:

- `local` — no isolation; trust your host
- `docker` — container boundary; combine with read-only volumes
- `ssh` — remote user; isolate from your laptop
- `modal` / `daytona` — cloud sandboxes with hibernation
- `singularity` — HPC-grade container on shared clusters

Regardless of backend, the agent's CWD is constrained to the configured
`terminal.cwd`; escapes require explicit approval.

Python code execution (`tools/code_execution_tool.py`) runs in a
subprocess with its own working directory and an RPC bridge; it does not
share the agent's process state.

## 4 · Memory-injection defences

External memory providers can be used to smuggle instructions into the
system prompt. Hermes's defences:

- Every memory-recalled chunk is wrapped in `<memory-context>` fences
  before injection.
- The system prompt instructs the model to treat these as **context**,
  not commands.
- `agent/memory_manager.sanitize_context` strips the fence markers from
  any model output so the agent cannot be tricked into re-emitting them
  as its own memory.

## 5 · Skills and MCP supply chain

Installed skills and MCP servers run with full tool privileges. Guards:

- `tools/skills_guard.py` — pattern scan before install (suspicious code,
  network calls, obfuscated strings, unpinned deps)
- `tools/osv_check.py` — OSV malware database lookup for any package a
  skill declares
- Skills Hub publishes signed manifests; the installer verifies provenance
- MCP servers are opt-in per config file

When in doubt, review the skill or MCP server code before `hermes skills
install`.

## 6 · Supply-chain auditing in CI

[`.github/workflows/supply-chain-audit.yml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.github/workflows/supply-chain-audit.yml)
runs on every PR and scans the diff for:

- `.pth` files (Python path injection)
- `base64 + exec` combos
- `subprocess` with encoded commands
- Unexpected network calls
- `setup.py`/`pyproject.toml` install hooks
- `marshal` / `pickle` of user-controlled data
- Workflow file changes (fails hard on CI changes that look suspicious)
- Dockerfile changes
- Unpinned GitHub Actions

Findings post as PR comments. **CRITICAL** findings fail the job.

## 7 · Gateway access control

The messaging gateway is reachable by anyone who can message the bot, so
access is gated at the adapter level:

- Per-platform `ALLOWED_USERS` env vars
- `GATEWAY_ALLOW_ALL_USERS=false` by default
- First-time pairing flows (QR scan, shared secret, admin approval)
- Home channel restrictions for scheduled output

See [[12-Gateway-And-Messaging]] for the full pairing model.

## 8 · Container hardening

The Docker image runs as non-root user `hermes` (UID 10000). The
entrypoint (`docker/entrypoint.sh`) uses `gosu` for safe privilege
transitions. Persistent data lives at `/opt/data`; everything else is
ephemeral.

## 9 · Reporting vulnerabilities

Do **not** open a public GitHub issue for security problems. Follow
[`SECURITY.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/SECURITY.md):
email `security@nousresearch.com` with the details.

## Pitfalls

- **`local` terminal backend + approval disabled** is the most common
  foot-gun. If you trust the model completely, you still don't trust
  every string the model generates — keep approvals on.
- **Over-permissive `approval.patterns`.** Allowlisting `rm -rf` broadly
  defeats the purpose; scope patterns to directories.
- **Skills from strangers.** `tools/skills_guard.py` is a sieve, not a
  firewall. Read the skill's `SKILL.md` and any scripts before installing.
- **External memory.** Anything the agent learns gets shipped to the
  provider. For private conversations, use `builtin` or `holographic`.

## Source of truth

- [`SECURITY.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/SECURITY.md)
- [`tools/approval.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/approval.py)
- [`agent/redact.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/redact.py)
- [`tools/skills_guard.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/skills_guard.py)
- [`tools/osv_check.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/osv_check.py)
- [`.github/workflows/supply-chain-audit.yml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.github/workflows/supply-chain-audit.yml)

## See also

- [[09-Terminal-Backends]]
- [[10-Skills-System]]
- [[11-Memory-System]]
- [[14-MCP-Integration]]
