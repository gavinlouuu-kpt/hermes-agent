# 04 · Repository Layout

A map of every top-level directory in the repo. When you want to know
"where does X live?", start here.

## Top-level files

| File | Purpose |
|---|---|
| `run_agent.py` | **Core `AIAgent` class** and `run_conversation` loop. The single class every interface calls. |
| `cli.py` | Interactive `prompt_toolkit` REPL implementation. Works with `hermes_cli/`. |
| `model_tools.py` | Orchestration: `get_tool_definitions()` / `handle_function_call()` bridging the registry to the LLM API. |
| `toolsets.py` | Logical groupings of tools (web, terminal, file, research, full_stack, …). |
| `toolset_distributions.py` | Distribution logic for RL toolset sampling. |
| `hermes_state.py` | SQLite session store with FTS5 full-text search. |
| `hermes_constants.py` | Global constants: `get_hermes_home()`, platform detection (Termux, WSL, Docker). |
| `hermes_logging.py` | Centralized logging with session-context tags. |
| `hermes_time.py` | Time utilities. |
| `utils.py` | Atomic JSON/YAML writes, env var helpers. |
| `batch_runner.py` | Parallel trajectory generation for training datasets. |
| `trajectory_compressor.py` | Trajectory compression for RL/dataset pipelines. |
| `mini_swe_runner.py` | Small SWE task runner (data generation). |
| `mcp_serve.py` | Standalone MCP server entrypoint wrapping Hermes tools. |
| `rl_cli.py` | CLI for RL training workflows. |
| `setup-hermes.sh` | Contributor setup script — uv, venv, `.[all]`, symlink. |
| `pyproject.toml` | Package metadata, deps, extras, entry points. |
| `package.json` | Node deps for browser tools + WhatsApp bridge. |
| `flake.nix` | Nix/NixOS package + dev shell. |
| `Dockerfile` | Multi-arch container build (Debian + Python 3.13 + uv). |
| `README.md` / `CONTRIBUTING.md` / `SECURITY.md` / `AGENTS.md` | Human docs. |
| `RELEASE_v*.md` | Per-release change logs. |
| `LICENSE` | MIT. |

## Top-level directories

### `agent/` — Agent internals
30+ focused modules extracted from `run_agent.py`:

- `prompt_builder.py` — system prompt assembly (identity, platform hints, skills index, context files)
- `context_compressor.py` — token-budget-driven history compression
- `auxiliary_client.py` — vision / summarization / embeddings multi-provider client
- `credential_pool.py` — multi-key failover (round-robin, least-used, random, fill-first)
- `memory_manager.py` / `memory_provider.py` — memory orchestration + plugin loader
- `model_metadata.py` — context lengths, token estimation, provider capabilities
- `anthropic_adapter.py` / `bedrock_adapter.py` / `gemini_cloudcode_adapter.py` — provider-specific adapters
- `display.py` — spinners, tool preview rendering, emoji selection
- `error_classifier.py` — classify LLM API errors for failover decisions
- `redact.py` — scrub secrets from logs and responses
- `skill_utils.py` / `skill_commands.py` — skill discovery, parsing, invocation
- `smart_model_routing.py` — provider selection hints
- `subdirectory_hints.py` — terminal CWD hints
- `trajectory.py` — scratchpad + trajectory persistence
- `usage_pricing.py` — token cost estimation
- `insights.py` — session analytics

### `hermes_cli/` — CLI frontend
Everything the `hermes` binary does before the conversation starts:

- `main.py` — entry point dispatched by `[project.scripts]`
- `config.py` — load/save `~/.hermes/config.yaml`, schema, migration
- `setup.py` — interactive setup wizard
- `auth.py` — OAuth + API key resolution, Nous Portal flow
- `commands.py` — slash-command registry + autocomplete
- `callbacks.py` — terminal callbacks: approval prompts, sudo, clarify
- `models.py` — OpenRouter / provider model catalog
- `doctor.py` — `hermes doctor` diagnostics
- `banner.py` — ASCII welcome banner
- `skin_engine.py` — theme/visual customization
- `skills_hub.py` — Skills Hub CLI interface
- `backup.py` / `clipboard.py` / `env_loader.py` — utilities
- `auth_commands.py` / `claw.py` — auth subcommands + OpenClaw migration
- `web_server.py` — FastAPI backend for the embedded dashboard
- `web_dist/` — built Vite bundle (packaged in the wheel)

### `tools/` — 40+ tools + environments
Every file is a self-registering tool module (see [[08-Tool-System]]).

Representative tools:

- `registry.py` — central `ToolRegistry` singleton
- `approval.py` — dangerous-command detection + approval gating
- `terminal_tool.py` — shell execution across 6 backends
- `file_tools.py` / `file_operations.py` — read/write/search/patch
- `web_tools.py` — Exa / Firecrawl / Parallel search and extract
- `browser_tool.py` — Browserbase / Camofox browser automation
- `vision_tools.py` — multimodal image analysis
- `code_execution_tool.py` — Python sandbox with RPC bridge
- `delegate_tool.py` — spawn subagents in parallel
- `session_search_tool.py` — FTS5 session search + LLM summarization
- `memory_tool.py` — MEMORY.md / USER.md CRUD
- `skills_tool.py` / `skill_manager_tool.py` / `skills_hub.py` / `skills_guard.py` — skills pipeline
- `cronjob_tools.py` — schedule management
- `mcp_tool.py` — MCP server client
- `tts_tool.py` / `voice_mode.py` / `transcription_tools.py` — speech
- `image_generation_tool.py` — FAL / FLUX image gen
- `homeassistant_tool.py` — smart-home control
- `mixture_of_agents_tool.py` — ensemble MoA
- `process_registry.py` — background processes
- `rl_training_tool.py` — in-agent RL utilities
- `send_message_tool.py` — gateway message delivery
- `osv_check.py` — malware scanning for packages
- `checkpoint_manager.py` — browser checkpoints
- `todo_tool.py` — todo list
- `environments/` — terminal backends (`base`, `local`, `docker`, `ssh`, `modal`, `daytona`, `singularity`)

### `gateway/` — Messaging gateway
- `run.py` — `GatewayRunner` orchestrator, per-session `AIAgent` cache (LRU + TTL)
- `config.py` — platform configs (Telegram, Discord, Slack, …)
- `session.py` — session store + context prompts
- `delivery.py` — message delivery routing
- `pairing.py` — multi-user session pairing / DM allowlist
- `channel_directory.py` — user/channel maps
- `stream_consumer.py` — async message-stream handling
- `hooks.py` / `builtin_hooks/` — webhook integration
- `platforms/` — adapters for telegram, discord, slack, whatsapp, signal, matrix, mattermost, dingtalk, feishu, qqbot, email, sms, homeassistant

### `ui-tui/` — React/Ink TUI (TypeScript)
Experimental frontend. `entry.tsx` + `app.tsx`, `gatewayClient.ts` bridges to
a Python subprocess via JSON-RPC. Ink components in `components/`.

### `tui_gateway/` — Python RPC backend for the Ink TUI
- `entry.py` — stdio entry point
- `server.py` — RPC handlers
- `render.py` — Rich/ANSI rendering
- `slash_worker.py` — persistent CLI subprocess

### `web/` — Vite dashboard sources
Build output ends up under `hermes_cli/web_dist/` and is served by
`hermes_cli/web_server.py`.

### `acp_adapter/` — IDE integration
Agent Client Protocol server used by Zed, VS Code, and JetBrains plugins.

### `skills/` — Bundled skills (26 categories)
Procedural memory bundles shipped with the default install. Each
sub-directory contains one or more `SKILL.md` files and helper scripts.
Example categories: `research/`, `creative/`, `productivity/`, `github/`,
`data-science/`, `software-development/`, `devops/`, `mlops/`, `media/`,
`leisure/`, `email/`, `smart-home/`, `apple/`, `mcp/`, `autonomous-ai-agents/`,
`red-teaming/`.

### `optional-skills/` — Opt-in skills (14 categories)
Not activated by default. Install with `hermes skills install …`. Includes
`blockchain/`, `communication/`, `health/`, `security/`, `migration/`
(OpenClaw → Hermes), and more `research/` / `mlops/` / `devops/` bundles.

### `plugins/` — Plugin system
- `context_engine/` — context-file processing plugin
- `memory/` — memory provider plugins (Holographic, Honcho, Hindsight, Supermemory, Mem0, RetainDB, OpenViking, ByteRover)
- `example-dashboard/` — template

### `cron/` — Scheduler
- `jobs.py` — job definitions
- `scheduler.py` — async scheduler loop
- Plus delivery + config glue used by `tools/cronjob_tools.py`

### `environments/` — RL training
- `hermes_base_env.py` — `HermesAgentBaseEnv` (Atropos integration base)
- `agentic_opd_env.py` — operational-design environment
- `web_research_env.py` — web research RL task
- `agent_loop.py` — `HermesAgentLoop` (multi-turn tool-calling runner)
- `tool_context.py` — reward-function snapshot
- `hermes_swe_env/`, `terminal_test_env/`, `benchmarks/`, `tool_call_parsers/`

### `tinker-atropos/` — Git submodule
Optional Atropos RL library. Initialise with
`git submodule update --init tinker-atropos`.

### `tests/` — pytest suite
Organised to mirror the code: `run_agent/`, `agent/`, `cli/`, `gateway/`,
`hermes_cli/`, `plugins/`, `environments/`, `integration/`, `e2e/`,
`honcho_plugin/`, `cron/`. Shared fixtures in `fakes/` and `conftest.py`.

### `website/` — Docusaurus docs site
Source for [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).
Built in CI (`.github/workflows/deploy-site.yml`) and deployed to GitHub
Pages + Vercel.

### `docs/` — Specs and design notes
- `honcho-integration-spec.md`
- `acp-setup.md`
- `specs/`, `skins/`, `plans/`, `migration/`

### `scripts/` — Install + utility scripts
`install.sh`, `install.ps1`, `install.cmd`, `build_skills_index.py`,
`contributor_audit.py`, `discord-voice-doctor.py`, `hermes-gateway`,
`kill_modal.sh`, `release.py`, `run_tests.sh`, `sample_and_compress.py`,
`whatsapp-bridge/` (Node.js Baileys bridge).

### `docker/` — Container support
Dockerfile lives at the repo root; `docker/` holds the entrypoint
(`entrypoint.sh`), compose templates, and backend-specific Dockerfiles.

### `nix/` — Nix/NixOS packaging
`packages.nix`, `nixosModules.nix`, `checks.nix`, `devShell.nix`,
`python.nix`, `tui.nix`, `web.nix`.

### `packaging/` — OS packaging
Homebrew formulae, Debian/Alpine build specs.

### `assets/` — Images
Banner graphics and logos used in `README.md` and the website.

### `datagen-config-examples/` — Data generation configs
Template YAMLs for RL trajectory generation.

### `.github/` — GitHub-native config
- `workflows/` — CI jobs (see [[19-CI-And-Releases]])
- `AUTHOR_MAP` — contributor attribution
- Issue/PR templates

## Source of truth

- [`pyproject.toml`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/pyproject.toml) — `[tool.setuptools.packages.find]` enumerates the Python packages
- [`AGENTS.md`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/AGENTS.md) — architectural narrative
- Directory contents on `main` in the repo

## See also

- [[05-Architecture]]
- [[08-Tool-System]]
- [[12-Gateway-And-Messaging]]
- [[21-Extending-Hermes]]
