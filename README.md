# autoresearch-labs

`autoresearch-labs` is a small workspace for studying and running Andrej Karpathy's `autoresearch` and `nanochat` side by side.

This repository is not a separate framework. It is a curated lab repo that:

- pins the upstream projects as Git submodules
- keeps analysis notes in one place
- makes it easier to compare the autonomous research loop with the parent training stack

The canonical implementation details still live in the upstream repos inside `source/`.

## Included projects

| Path | Upstream | Purpose |
| --- | --- | --- |
| `source/autoresearch` | `karpathy/autoresearch` | Single-GPU autonomous research loop where an agent iterates on `train.py` under guidance from `program.md`. |
| `source/nanochat` | `karpathy/nanochat` | Minimal end-to-end LLM training and chat stack that `autoresearch` is derived from. |

## Why this repo exists

- Keep a reproducible local snapshot of both upstream projects.
- Read the two codebases together instead of in isolation.
- Store writeups and analysis without modifying the upstream repositories.

## Pinned snapshots

At the time of this snapshot, the submodules point to:

- `source/autoresearch`: `c2450ad`
- `source/nanochat`: `f068604`

## Repository layout

```text
.
├── README.md
├── LICENSE
├── medium-autoresearch-analysis.md
├── medium-autoresearch-analysis-en.md
└── source
    ├── autoresearch
    └── nanochat
```

## Getting started

### 1. Initialize submodules

Clone with submodules:

```bash
git clone --recurse-submodules <repo-url>
cd autoresearch-labs
```

If you already cloned the repository without submodules:

```bash
git submodule update --init --recursive
```

### 2. Install `uv`

Both subprojects use [`uv`](https://docs.astral.sh/uv/) for environment and dependency management.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 3. Choose the project you want to run

#### Run `autoresearch`

`autoresearch` is the smaller, research-loop-focused setup. It expects a single NVIDIA GPU for the default workflow.

```bash
cd source/autoresearch
uv sync
uv run prepare.py
uv run train.py
```

Use the upstream guide in `source/autoresearch/README.md` for the full workflow and `program.md`-driven agent loop.

#### Explore `nanochat`

`nanochat` is the broader training stack that covers tokenization, pretraining, finetuning, evaluation, inference, and a chat UI.

GPU setup:

```bash
cd source/nanochat
uv sync --extra gpu
```

CPU-only setup:

```bash
cd source/nanochat
uv sync --extra cpu
```

From there, follow `source/nanochat/README.md` for the path you want, such as `runs/speedrun.sh`, `runs/miniseries.sh`, or `runs/runcpu.sh`.

## Notes and analysis

- `medium-autoresearch-analysis.md`: Korean long-form analysis of `autoresearch`, `nanochat`, and the AI-native research loop.
- `medium-autoresearch-analysis-en.md`: English version of the same analysis.

## Working style

This repo is best treated as a lab notebook around upstream code:

- read the root `README` for orientation
- use the subproject `README`s as the source of truth for execution details
- keep local notes and comparisons at the root level

## License

The root repository is MIT-licensed. The included upstream projects also ship with their own MIT licenses; see each subproject directory for details.
