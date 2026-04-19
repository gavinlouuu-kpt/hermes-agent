# 22 · Contributing

Hermes welcomes contributions. This page is a distillation of
[`CONTRIBUTING.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/CONTRIBUTING.md)
and
[`AGENTS.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/AGENTS.md);
read those for the complete version.

## Getting set up

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh       # installs uv, venv, .[all]
./hermes                # sanity check
pytest tests/ -q        # run the suite
```

See [[02-Installation]] for manual setup.

## The development loop

1. Pick an issue or file one. `good-first-issue` label is a good start.
2. Create a feature branch from `main`.
3. Make focused changes (one concern per PR).
4. Run tests: `pytest tests/ -q`.
5. Run the affected tool/skill/platform in an actual Hermes session.
6. Commit with descriptive messages.
7. Open a PR. CI will exercise unit + e2e tests, Nix build, and
   supply-chain audit.

## Code conventions

- **Python 3.11+.** Type hints encouraged; not strict throughout.
- **No new top-level dependencies** without a justification in the PR
  description — the core deps in `pyproject.toml` are pinned to limit
  supply-chain risk.
- **Small, focused modules.** See `agent/` for the pattern: each file
  owns one concern.
- **Tests alongside code.** New tools, skills, and platform adapters
  need at least one pytest under `tests/`.
- **Logging uses `hermes_logging`** so session context is preserved.
- **Secrets via `agent/redact.py`** — never raw-print provider keys.

## Where the work is

- **Tools**: `tools/` — new APIs, integrations, utilities
- **Skills**: `skills/`, `optional-skills/` — SKILL.md bundles
- **Gateway platforms**: `gateway/platforms/` — new messaging integrations
- **Memory providers**: `plugins/memory/` — new memory backends
- **MCP integration**: `tools/mcp_tool.py`, `optional-skills/mcp/`
- **RL**: `environments/` — new training environments
- **Docs**: `website/docs/` — Docusaurus content (this wiki is a separate,
  in-repo distillation, not the canonical user docs)

## Running tests locally

```bash
pytest tests/ -q                              # default: skips integration
pytest tests/run_agent/ -xvs                  # specific area, verbose
pytest tests/ -m integration                  # integration only (needs keys)
pytest tests/e2e/ -v                          # end-to-end
pytest tests/tools/test_my_tool.py -xvs       # single file
```

Integration tests require provider API keys in `~/.hermes/.env` or the
environment.

## Pre-submit checklist

- [ ] Tests pass: `pytest tests/ -q`
- [ ] The feature works in a manual run: `./hermes` → try it
- [ ] New/changed env vars added to `.env.example`
- [ ] New config keys added to `cli-config.yaml.example`
- [ ] Release note snippet drafted (include in PR description)
- [ ] No new secrets committed (check `git diff --stat` for `.env*` files)
- [ ] Supply-chain audit clean — re-review if CI flags anything

## Commit & PR style

- Imperative mood: "Add X", not "Added X".
- Reference the relevant issue: `Fixes #1234`.
- One topic per PR. Refactors separate from features.
- Keep PR descriptions short but complete: what changed, why, how to
  test.

## CLAUDE.md / AGENTS.md

Many contributors use Claude Code or similar agents to work on this
repo. `AGENTS.md` is written for those agents — it summarizes the
architecture, coding conventions, and common tasks. If you use an AI
coding assistant, point it at that file.

## Community

- **Discord**: <https://discord.gg/NousResearch>
- **Issues**: <https://github.com/NousResearch/hermes-agent/issues>
- **Discussions**: <https://github.com/NousResearch/hermes-agent/discussions>

## Security disclosures

Email `security@nousresearch.com`. Do **not** file public issues for
security problems. See
[`SECURITY.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/SECURITY.md).

## Attribution

Contributor attribution is maintained in
[`.github/AUTHOR_MAP`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/.github/AUTHOR_MAP).
Release notes credit contributors by username.

## Source of truth

- [`CONTRIBUTING.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/CONTRIBUTING.md)
- [`AGENTS.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/AGENTS.md)
- [`SECURITY.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/SECURITY.md)
- [`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml)

## See also

- [[19-CI-And-Releases]]
- [[21-Extending-Hermes]]
