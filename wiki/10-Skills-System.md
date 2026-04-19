# 10 · Skills System

A **skill** is a reusable unit of procedural memory: a Markdown file (plus
optional helper scripts) that teaches the agent how to do a specific task.
Hermes ships with 26+ bundled skills, supports 14+ optional skills, and can
create new skills autonomously after complex tasks. Skills are compatible
with the open [agentskills.io](https://agentskills.io) format.

## Anatomy of a skill

Every skill lives in its own directory and contains at minimum a
`SKILL.md`:

```
skills/research/arxiv/
├── SKILL.md            # front-matter + instructions
├── search.py           # optional helper script
├── examples/           # optional worked examples
└── README.md           # optional human-facing notes
```

`SKILL.md` starts with YAML front-matter:

```markdown
---
name: research:arxiv
description: Search arXiv, fetch papers, summarize abstracts.
triggers:
  - "search arxiv"
  - "find a paper about"
toolsets_required: [web]
scripts:
  - search.py
---

# Instructions to the model

1. Use `web_search` to query arXiv (site:arxiv.org).
2. For each match, ...
```

Front-matter is parsed by
[`agent/skill_utils.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/skill_utils.py)
and used to build the skills index shown to the model.

## The skills index

On every turn, `agent/prompt_builder.py` injects a **skills index** into the
system prompt — a compact table of available skills with their
descriptions and trigger phrases. When a user's message matches a trigger,
the model knows to invoke the skill.

Skills can be invoked explicitly via `/` slash commands:

```
/research:arxiv   "self-improving agents"
```

Or implicitly — the model decides to load a skill based on context.

## Bundled skill categories

Under [`skills/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/skills):

`apple/`, `autonomous-ai-agents/`, `creative/` (image, music, video, p5js,
manim, Excalidraw), `data-science/`, `devops/`, `email/`, `github/`,
`leisure/`, `mcp/`, `media/` (YouTube content), `mlops/` (vLLM, inference),
`productivity/` (Google Workspace, PowerPoint, OCR), `red-teaming/`,
`research/` (arxiv, polymarket), `smart-home/`, `software-development/`.

## Optional skills

Under [`optional-skills/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/optional-skills).
Not auto-loaded; install with `hermes skills install <name>`.
Includes `blockchain/` (Solana, Base), `communication/`, `health/`,
`security/`, `migration/` (OpenClaw → Hermes), deeper `research/`,
`mlops/` (Slime, Accelerate, SAE Lens, Chroma, Flash Attention, Guidance,
Instructor, HuggingFace Tokenizers, vLLM), `devops/` (docker-management),
`productivity/` (Memento flashcards, Telephony, Canvas), `creative/`
(meme-generation, TouchDesigner MCP, Blender MCP).

## Skills Hub

[Skills Hub](https://agentskills.io) is the community registry. Hermes
integrates with it through:

- [`tools/skills_hub.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/skills_hub.py) — API client
- [`hermes_cli/skills_hub.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/hermes_cli/skills_hub.py) — CLI subcommands
- [`tools/skills_guard.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/skills_guard.py) — security scanning before install

Commands:

```bash
hermes skills list             # installed
hermes skills search <query>   # Hub search
hermes skills install <name>   # download + guard + install
hermes skills publish <dir>    # publish your own
hermes skills update           # refresh installed skills
```

## Autonomous skill creation

After complex tasks the agent can suggest creating a new skill — it
summarises the workflow, writes a `SKILL.md` under `~/.hermes/skills/`,
and asks the user for confirmation. This is implemented via the
`skills_tool.py` + `skill_manager_tool.py` combo.

## Security scanning

Installed skills run with tool access, so they're scanned before first use:

- Dangerous shell patterns
- Network calls to unexpected hosts
- Obfuscated code (base64, `exec`, marshal/pickle)
- Unpinned or suspicious dependencies

`tools/skills_guard.py` implements these checks; `tools/osv_check.py`
cross-references package names against the OSV malware database.

## Helper scripts

A skill's `scripts/` directory is added to `$PATH` when the skill is
loaded so the model can invoke helpers by name. Scripts run through the
configured terminal backend and respect the approval gate.

## Authoring (short version)

1. `hermes skills new <name>` — scaffolds a directory under
   `~/.hermes/skills/<name>/`.
2. Edit `SKILL.md` front-matter (name, description, triggers,
   toolsets_required).
3. Write the instructions in the body — be declarative, include a short
   worked example.
4. Add helper scripts under `scripts/` if needed.
5. Test with `/your:skill` in the CLI.
6. Optional: `hermes skills publish .` to put it on the Hub.

Full walkthrough in [[21-Extending-Hermes]].

## Pitfalls

- **Skills are Markdown, not code.** Keep instructions declarative; the
  model, not the skill, decides which tools to call.
- **Don't embed secrets.** Front-matter is indexed into the system prompt
  of every turn.
- **Match triggers carefully.** Overbroad triggers can flood the index and
  distract the model.
- **Helper scripts run with full tool privileges.** Review third-party
  skills before installing — the guard catches obvious issues but not
  sophisticated ones.

## Source of truth

- [`skills/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/skills)
- [`optional-skills/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/optional-skills)
- [`tools/skills_tool.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/skills_tool.py)
- [`tools/skills_hub.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/skills_hub.py)
- [`tools/skills_guard.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/tools/skills_guard.py)
- [`agent/skill_utils.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/agent/skill_utils.py)
- [agentskills.io](https://agentskills.io) — open standard

## See also

- [[11-Memory-System]]
- [[14-MCP-Integration]]
- [[17-Security-Model]]
- [[21-Extending-Hermes]]
