# 20 · RL and Trajectories

Hermes has first-class support for generating agent **trajectories**
(sequences of tool calls + results + rewards) suitable for reinforcement
learning on tool-use models. The RL stack integrates with
[Atropos](https://github.com/NousResearch/atropos) and
[Tinker](https://github.com/thinking-machines-lab/tinker), both optional
and pinned via `pyproject.toml` extras.

## Key files

- [`batch_runner.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/batch_runner.py) — parallel trajectory generation
- [`trajectory_compressor.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/trajectory_compressor.py) — compress trajectories for training
- [`rl_cli.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/rl_cli.py) — CLI for RL workflows
- [`mini_swe_runner.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/mini_swe_runner.py) — small SWE task runner
- [`environments/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/environments) — Atropos environment adapters
- [`toolset_distributions.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/toolset_distributions.py) — toolset sampling for training
- [`tinker-atropos/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tinker-atropos) — git submodule (optional)

## Install the RL extra

```bash
git submodule update --init tinker-atropos
uv pip install -e ".[rl]"
```

`.[rl]` pulls in pinned versions of `atroposlib` and `tinker` (both
direct-from-git), plus `fastapi`, `uvicorn`, and `wandb` for the training
server.

## The environment abstraction

`environments/hermes_base_env.py` defines `HermesAgentBaseEnv`, the
Atropos-compatible base class:

- `reset()` — initialize a task, return the first observation
- `step(action)` — execute a tool call, return (obs, reward, done, info)
- Internally runs `environments/agent_loop.py`'s `HermesAgentLoop` for
  multi-turn tool-calling
- Manages a per-environment server, toolset resolution, and trajectory
  collection

Concrete environments:

- [`environments/agentic_opd_env.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/environments/agentic_opd_env.py) — operational-design tasks
- [`environments/web_research_env.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/environments/web_research_env.py) — web research
- [`environments/hermes_swe_env/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/environments/hermes_swe_env) — software-engineering tasks
- [`environments/terminal_test_env/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/environments/terminal_test_env) — terminal-based tasks
- [`environments/benchmarks/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/environments/benchmarks) — standard benchmark adapters

## The training server

Environments expose an HTTP endpoint consumed by Atropos trainers. The
default URL is `http://localhost:8080` (override with `RL_API_URL`). The
server:

- Receives model responses from the trainer
- Dispatches tool calls via the Hermes registry
- Returns observations + rewards
- Checkpoints trajectories to disk

## Batch trajectory generation

For dataset building (not on-policy training), use `batch_runner.py`:

```bash
python batch_runner.py \
    --config datagen-config-examples/web_research.yaml \
    --num-episodes 10000 \
    --concurrency 32 \
    --output data/trajectories.jsonl
```

This spawns many `AIAgent` instances in parallel, each running until
completion (or failure), and writes trajectories as JSONL. Config
templates live in
[`datagen-config-examples/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/datagen-config-examples).

## Trajectory format

Each record is a JSON object with:

```json
{
  "session_id": "01HK7A…",
  "task": "research: find three recent papers on …",
  "messages": [
    {"role": "system", "content": "…"},
    {"role": "user", "content": "…"},
    {"role": "assistant", "tool_calls": [...]},
    {"role": "tool", "tool_call_id": "…", "content": "…"},
    ...
  ],
  "tool_calls_made": 12,
  "input_tokens": 4321,
  "output_tokens": 987,
  "reward": 0.78,
  "info": {...}
}
```

## Compression for training

Raw trajectories are large — many of them exceed model context limits.
`trajectory_compressor.py` reduces them to a trainable form:

- Drops intermediate tool results that weren't used in the final answer
- Summarizes long tool outputs
- Truncates verbose reasoning
- Preserves tool-call structure (the signal you're training on)

Run via `scripts/sample_and_compress.py`:

```bash
python scripts/sample_and_compress.py \
    --input data/trajectories.jsonl \
    --output data/trajectories.compressed.jsonl \
    --max-tokens 16000
```

## Tool-set sampling

For diversity, training runs can randomize which toolset is available
per episode. `toolset_distributions.py` defines named distributions:

```python
sample_toolsets_from_distribution("balanced")
sample_toolsets_from_distribution("web_heavy")
sample_toolsets_from_distribution("full_stack")
```

Referenced from environment configs to prevent the model from overfitting
to a single toolset.

## Training with Tinker

[Tinker](https://github.com/thinking-machines-lab/tinker) is Thinking
Machines Lab's RL toolkit. The integration:

- `tinker-atropos/` submodule installs both Tinker and Atropos
- `rl_cli.py` wraps common training commands
- `WANDB_API_KEY` enables experiment tracking

Typical flow:

1. Generate or curate a dataset with `batch_runner.py`.
2. Start an environment server.
3. Run Tinker's SFT or RL trainer pointing at the server.
4. Evaluate new checkpoints by swapping `model.default` in
   `config.yaml`.

## Evaluation benchmarks

`environments/benchmarks/` contains adapters for standard benchmarks
(e.g. SWE-bench-like workloads via `hermes_swe_env/`). `yc-bench` is
pinned as an optional extra for Y Combinator benchmark compatibility.

## Pitfalls

- **RL submodule drift.** The `tinker-atropos/` submodule is pinned to a
  specific commit. After `git pull`, run
  `git submodule update --init --recursive`.
- **Trajectory storage.** A million trajectories is tens of GB.
  Compression is not optional at scale.
- **Tool-call correctness.** A model that generates syntactically invalid
  tool calls will produce trajectories your trainer can't learn from.
  Keep tool schemas simple; see [[08-Tool-System]].
- **Reward hacking.** Make sure rewards in your environment can't be
  trivially gamed by the agent (e.g. "wrote the output file" without
  validating content).

## Source of truth

- [`batch_runner.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/batch_runner.py)
- [`trajectory_compressor.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/trajectory_compressor.py)
- [`environments/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/environments)
- [`rl_cli.py`](https://github.com/gavinlouuu-kpt/hermes-agent/blob/main/rl_cli.py)
- [`tinker-atropos/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/tinker-atropos)
- [`datagen-config-examples/`](https://github.com/gavinlouuu-kpt/hermes-agent/tree/main/datagen-config-examples)
- [Atropos](https://github.com/NousResearch/atropos) · [Tinker](https://github.com/thinking-machines-lab/tinker)

## See also

- [[06-Agent-Loop]]
- [[08-Tool-System]]
- [[22-Contributing]]
