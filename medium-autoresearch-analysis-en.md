# Autoresearch: Why Karpathy Turned `program.md` Into a Research Operating System

Subtitle: A deep analysis of `autoresearch`, `nanochat`, and the design of an AI-native research loop

## TL;DR

`autoresearch` is not a giant agent framework. It is almost the opposite. Instead of making research automation more complex, Karpathy compresses the loop into a very small system: fixed data preparation in `prepare.py`, a single editable training file in `train.py`, and a natural-language operating manual in `program.md`.

That design choice matters. The project is not trying to make agents capable of doing everything. It is trying to leave only the parts of research that agents can do well. And that narrower design appears to work. Improvements found through `autoresearch` were later reflected in `nanochat`, where the reported time to GPT-2-grade capability dropped from 2.02 hours to 1.80 hours on the leaderboard, about an 11% improvement.

This article is a source-based analysis of `autoresearch`, `nanochat`, and Karpathy's public notes around the project. The goal is to explain why this repo is more than a clever demo, and why it may be an early prototype for how AI starts eating the research workflow itself.

## Scope and Sources

This article was written on March 14, 2026, after cloning and reading the following versions locally:

- `autoresearch` HEAD: `c2450ad`
- `nanochat` HEAD: `f068604`
- `nanochat` commit that reflects autoresearch round 1: `6ed7d1d`

Primary sources:

- X post: [karpathy/status/2031135152349524125](https://x.com/karpathy/status/2031135152349524125)
- Text mirror: [xcancel.com/karpathy/status/2031135152349524125](https://xcancel.com/karpathy/status/2031135152349524125)
- `autoresearch` repo: [github.com/karpathy/autoresearch](https://github.com/karpathy/autoresearch)
- `nanochat` repo: [github.com/karpathy/nanochat](https://github.com/karpathy/nanochat)
- Round 1 integration commit: [nanochat commit 6ed7d1d](https://github.com/karpathy/nanochat/commit/6ed7d1d82cee16c2e26f45d559ad3338447a6c1b)

## Why This Project Matters

Most agent demos stop at capability theater: browser use, tool calls, file edits, code generation. `autoresearch` is different. It does not try to make an agent look like a human researcher. It redesigns research itself into a shape that an agent can optimize.

The core move is simple:

1. Fix the objective to a single number. Here, that number is `val_bpb`.
2. Fix the experiment budget to a single time window. Here, it is 5 minutes of wall-clock training time.
3. Restrict the editable surface to a single file. Here, it is `train.py`.

That combination is unusually powerful. The agent no longer has to solve "how should I do research?" It only has to solve "what should I change in this file to get a lower `val_bpb` within 5 minutes?"

So `autoresearch` is not best understood as general intelligence being dropped into research. It is better understood as research being reformulated into an agent-friendly optimization problem. That is what makes it practical.

## What the Sources Actually Establish

Before interpreting the project, it is worth separating out the hard facts.

- The `autoresearch` README describes the project as giving an AI agent a small but real LLM training setup and letting it experiment autonomously overnight.
- The same README explicitly says this repo is a simplified single-GPU implementation of `nanochat`.
- `program.md` defines the operating protocol for the agent in natural language: create a branch, run a baseline, edit `train.py`, commit, train for 5 minutes, compare results, keep or discard, and log the run to `results.tsv`.
- The `nanochat` README adds an `autoresearch round 1` leaderboard entry and states that time to GPT-2-grade capability dropped from 2.02 hours to 1.80 hours.
- The `nanochat` commit `6ed7d1d` states that these improvements were developed by Claude running autonomously over about 2 days using `autoresearch`, and that Karpathy did not touch them.
- At the same time, in a March 9, 2026 X reply, Karpathy noted that parts of the earlier weekend version were not yet fully time-controlled and that some experiments were rejected by jointly considering loss and training time. So the public repo should not be treated as a perfectly exact snapshot of the round 1 internal loop.

Taken together, those points make the nature of the project fairly clear. `autoresearch` is both a constrained training-code sandbox for an agent and a lightweight autonomous research protocol.

## What Karpathy Was Probably Trying To Do

This section is interpretive, but the interpretation is anchored in the README, `program.md`, the `nanochat` integration commit, and the public post around the project.

I think Karpathy's intent operates on at least four levels.

### 1. Move the human from code author to research-organization designer

The `autoresearch` README does not frame the human as the person who keeps changing Python files. It frames the human as the person who "programs the `program.md` Markdown files." That phrase matters. It implies a shift in where research labor lives.

The human is no longer primarily the one editing the model over and over. The human becomes the one designing the protocol that the agent follows.

### 2. Redesign research so an agent can succeed at it

Karpathy does not give the agent unconstrained freedom. The editable file is basically `train.py`. The metric is basically `val_bpb`. The budget is basically 5 minutes. That is not "we built an AI researcher." It is closer to "we redesigned a research loop so an agent has a high chance of succeeding inside it."

This is a very practical choice. Agentic systems often fail not because they are too weak in some abstract sense, but because the task definition is too broad and the state space is too large. `autoresearch` cuts that down aggressively.

### 3. Improve a real upstream project, not just stage a demo

This project did not stop at a concept. The `nanochat` README ties `autoresearch round 1` to a concrete leaderboard improvement, and commit `6ed7d1d` says those changes came from roughly two days of autonomous search. So the real question was not "can an agent imitate research?" It was "can this setup materially improve an actual research codebase?"

### 4. Push humans out of the short experimental feedback loop

`program.md` tells the agent to establish a baseline, run experiments, commit, read metrics, keep or discard, and continue without stopping. That wording is not accidental. It fits the same picture Karpathy sketches publicly: agents collaborate, and humans contribute on the edges. The repo reads like an experiment in moving humans away from the center of the tight loop and into the design and interpretation layers around it.

Compressed into one line, the intent looks like this:

Karpathy was not mainly declaring that AI can replace researchers. He was testing whether research loops themselves can be redesigned into an interface that AI can keep running.

## The Core Idea of `autoresearch`

The real invention here is not a new model architecture. It is the separation of research into three layers.

### 1. The invariant execution layer: `prepare.py`

This layer fixes the rules of the experiment: data, tokenizer, evaluation, time budget, and the runtime utilities around them. The agent does not edit it.

### 2. The search layer: `train.py`

This is where the agent actually explores: architecture, initialization, optimizer, batch size, schedules, and training behavior.

### 3. The organization layer: `program.md`

This is not Python code that runs on the GPU. It is the operating system for the agent. It specifies branching, logging, revert policy, baseline handling, result recording, and continuation behavior.

Put differently:

- Python defines the model.
- Markdown defines the research organization.

When the README says "you are programming the `program.md` Markdown files," it is not rhetorical fluff. In this repo, `program.md` really is the lab's standard operating procedure.

## Project Structure: Small Repo, Large Intention

`autoresearch` is deliberately small. The practical core is almost entirely four files.

| File | Role | Primary editor |
|---|---|---|
| `prepare.py` | data download, tokenizer training, dataloader, BPB evaluation | human / fixed |
| `train.py` | model, optimizer, training loop | agent |
| `program.md` | experiment protocol, git loop, logging rules | human |
| `analysis.ipynb` | post-hoc analysis and visualization | human |

That structure matters because it narrows the editable surface while keeping almost all meaningful research levers inside `train.py`.

This is not just "constraint." It is careful product design. Too much freedom and the agent wanders. Too little and it cannot improve anything. `autoresearch` tries to sit in the middle.

## Code-Level Analysis 1: `prepare.py`

`prepare.py` is not just a setup script. It is closer to the constitution of the experiment.

### 1. Fixed constants

The file defines several crucial constants:

- `MAX_SEQ_LEN = 2048`
- `TIME_BUDGET = 300`
- `EVAL_TOKENS = 40 * 524288`
- `VOCAB_SIZE = 8192`
- Fixed validation shard: `shard_06542.parquet`

The two most important are `TIME_BUDGET` and the pinned validation shard.

`TIME_BUDGET = 300` changes the optimization target. The agent is no longer searching for the theoretically best model in the abstract. It is searching for the model that trains best on this hardware in 5 minutes. That means `autoresearch` implicitly optimizes both model quality and system efficiency at once.

Pinning validation to one shard makes runs directly comparable. This sacrifices some statistical rigor in exchange for speed and consistency, which is a deliberate trade in this repo.

### 2. Data download

The data comes from the Hugging Face dataset `karpathy/climbmix-400b-shuffle` in parquet shards. The user may download only some training shards, but the validation shard is always forced in.

That design reduces one source of experimental noise. It prevents the comparison target from drifting around with the sampling process and keeps a stable point of reference from run to run.

### 3. Tokenizer training

The tokenizer is trained with `rustbpe` and then wrapped into a `tiktoken.Encoding`. This stage matters more than it may first appear.

- The tokenizer is stored outside the repo in `~/.cache/autoresearch/`.
- A per-token byte-length lookup tensor is built and saved.
- Special tokens are given byte length 0.

That is all in service of the evaluation metric. `autoresearch` does not evaluate ordinary cross-entropy loss alone. It evaluates a byte-normalized metric, so vocabulary size changes are still comparable.

### 4. Dataloader design

`make_dataloader` implements BOS-aligned best-fit packing.

The logic is:

1. Every row starts with BOS.
2. From a buffer of documents, choose the largest document that fits in the remaining space.
3. If none fits, crop the shortest document to fill the row exactly.
4. Avoid padding and target 100% utilization.

This is research-relevant, not incidental. Compared with simpler padded batching, it reduces token waste and makes sequence boundaries more coherent by aligning rows to document starts. The tradeoff is cropping. Some original document structure is lost. But this is not a toy idea invented for `autoresearch`; `nanochat` uses the same general approach, so the repo is inheriting a battle-tested input pipeline and shrinking it down.

### 5. The evaluation function: `evaluate_bpb`

This function is the real constitution of the project.

- Fixed `MAX_SEQ_LEN`
- Fixed validation split
- Per-token cross-entropy normalized by byte length
- Special tokens excluded from both sums

The result is `bits per byte`, a vocabulary-size-invariant metric. That is a smart choice because it lets the agent compare runs more fairly even if tokenization-related changes are involved.

The downside is also real. Because validation is pinned to one shard, the system cannot fully rule out overfitting to the specific characteristics of that shard.

## Code-Level Analysis 2: `train.py`

`train.py` is the battlefield. For the agent, this is the file that gets read, patched, tested, committed, and sometimes reverted.

### 1. Runtime setup and dependencies

The top of the file is fairly aggressive:

- `PYTORCH_ALLOC_CONF=expandable_segments:True`
- CUDA capability is checked to choose a Flash Attention 3 kernel source
- `prepare.py` is imported for time budget, tokenizer, dataloader, and evaluation

One important nuance: the repo is not completely closed over its own code. Through the `kernels` package it pulls in external Flash Attention implementations. The README emphasizes simplicity and self-containment, but in a strict sense part of the execution path still depends on code outside the repo. That is a small but meaningful practical compromise.

### 2. Model structure

The model is a compressed version of `nanochat`'s pretraining GPT inside a single file.

Key traits:

- rotary embeddings
- RMSNorm-based `norm`
- untied embedding and `lm_head`
- `relu^2` MLP
- alternating value embeddings
- sliding-window attention
- Flash Attention 3
- residual scalar parameters `resid_lambdas` and `x0_lambdas`

This is not a weak GPT baseline. It is already a fairly strong and opinionated compact pretraining harness. In other words, `autoresearch` is not asking an agent to invent research from scratch. It is giving the agent a well-designed playground and asking it to search locally inside that playground.

### 3. `GPTConfig` and structural guardrails

`build_model_config(depth)` computes the model dimension from `depth * ASPECT_RATIO`, then rounds up to a multiple of `HEAD_DIM`.

That matters because shape consistency is one of the easiest ways for agents to break training code. `autoresearch` puts some of those guardrails into helper logic so the agent does not have to solve tensor geometry from first principles every time.

### 4. The attention block

`CausalSelfAttention` includes several features that stand out:

- `ve_gate` for value-embedding mixing
- rotary embedding followed by QK norm
- sliding-window attention patterning
- a Flash Attention 3 call for speed

One interesting implication is that the baseline is already researchy. `autoresearch` is not a project where a human hands the agent a blank GPT-2. A human has already designed a compact, high-performance research surface. The agent then searches on top of that.

### 5. MLP and initialization

The MLP uses `relu().square()`, matching `nanochat` and more recent Karpathy code patterns.

Initialization is also clearly intentional:

- embeddings use normal initialization
- some projections in attention and MLP are zero-initialized
- residual scalars get separate initialization
- the value-embedding gate starts in a neutral configuration

Those are exactly the kinds of surfaces an agent can profitably perturb. In fact, several of the round 1 changes later merged into `nanochat` touched gate behavior and initialization.

### 6. Optimizer: `MuonAdamW`

The densest part of the file is the optimizer section. For most readers, the right mental model is not the precise math, but the systems goal: this optimizer design is there to make the 5-minute loop as productive as possible.

`train.py` splits parameters into two broad regimes:

- AdamW for embeddings, unembedding, and scalar parameters
- Muon for 2D matrix parameters

Then it groups matrix parameters by shape and applies fused Muon updates.

That gives the agent a meaningful optimization surface. This is not just "you may change the optimizer." It is a setup where optimizer hyperparameters and group-specific treatment can plausibly move real results in a short time budget.

Also, `adamw_step_fused` and `muon_step_fused` are both `torch.compile`'d. That means the repo is leaning hard into execution efficiency.

That has two consequences:

1. Good: more useful training can fit inside 300 seconds.
2. Risky: if the agent changes structure too aggressively, compilation failures or performance regressions become easier to trigger.

So while the search space is simplified, the system itself is not especially simple.

### 7. Hyperparameter surface

The major knobs are all exposed as top-level constants:

- `ASPECT_RATIO`
- `HEAD_DIM`
- `WINDOW_PATTERN`
- `TOTAL_BATCH_SIZE`
- multiple learning rates
- `WEIGHT_DECAY`
- `DEPTH`
- `DEVICE_BATCH_SIZE`

This is much more agent-friendly than a more elaborate CLI or config system. The agent can patch values directly in context without having to manage flag plumbing across shell scripts and invocation layers.

That is not just a style preference. Agents are usually stronger at local code edits than at consistently managing a multi-file run configuration stack. `autoresearch` leans into that reality.

### 8. Time-based scheduling

This is one of the most important design choices in the whole repo.

`autoresearch` schedules by training time, not by step count.

- `progress = total_training_time / TIME_BUDGET`
- LR warmup and warmdown are tied to time progress
- the first 10 steps are excluded to avoid counting compile warmup

That is significant because it keeps runs comparable even when the agent changes model size, batch size, throughput, or kernel behavior. The hidden objective is not just "learn well." It is:

> Learn as well as possible in 300 seconds.

That compresses quality, throughput, kernel efficiency, memory fit, and scheduling into one practical target.

### 9. The training loop

The loop itself is small:

```text
prefetch batch
for 5 minutes:
  gradient accumulation
  schedule update
  optimizer step
  fail fast on NaN or explosion
end
evaluate val_bpb once
print summary
```

Important details:

- Gradient accumulation is used to hit `TOTAL_BATCH_SIZE`.
- If loss becomes NaN or exceeds 100, the run fails fast.
- Python GC is managed manually to reduce large pause spikes.
- The final summary prints `val_bpb`, `training_seconds`, `peak_vram_mb`, `mfu_percent`, and `num_params_M`.

That output format is not accidental. It lines up with the grep-based loop described in `program.md`. The Python code and the Markdown protocol are designed together.

## Code-Level Analysis 3: `program.md`

Honestly, the most important innovation in `autoresearch` may be `program.md`, not `train.py`.

This file tells the agent to:

- create a fresh branch
- run the baseline first
- modify only `train.py`
- commit before each experiment
- run `uv run train.py > run.log 2>&1`
- read key metrics with `grep`
- keep good changes and reset bad ones
- log keep, discard, and crash results to `results.tsv`
- never stop once the loop starts

That is effectively a finite-state machine for an autonomous research agent, written in natural language.

More importantly, it is not just "a prompt." It bundles together:

1. the objective
2. tool-usage constraints
3. state-transition rules
4. the memory and logging format

Many teams would instinctively implement this orchestration layer as Python. `autoresearch` moves it into Markdown. That makes the organization logic easier for a human to inspect and easier for a general coding agent to follow consistently.

This is beyond prompt engineering. It is closer to research-process engineering.

### Why `program.md` in Markdown matters

This question matters a lot. Many teams would have chosen YAML, JSON, or a Python orchestration framework. Instead, the public repo makes Markdown central. The README even describes `program.md` as a "super lightweight skill." That suggests it is being used as more than a document format.

First, Markdown is simultaneously readable by humans and legible to language models. For a human it is a document. For an LLM it is context. No translation layer is required.

Second, Markdown cleanly separates execution rules from model code. `train.py` becomes the model-search surface. `program.md` becomes the research-organization surface. That means a human can redesign the process without editing the training code itself.

Third, Markdown fits naturally into git. Diffs are readable, changes are reviewable, and the operating protocol itself becomes versioned. In that sense, `program.md` is not just a prompt. It is operational code that happens to be human-readable.

Fourth, Markdown is not tied to a single agent runtime. The README explicitly mentions Claude and Codex. That suggests the repo prefers a protocol that any competent coding agent can read over a highly specialized controller.

Fifth, Markdown is cheap to iterate on. If the agent is not behaving well, you can change the wording, the loop rules, the logging requirements, or the judgment criteria immediately. In the early stages of research automation, iteration speed matters more than architectural elegance. Markdown is excellent at iteration speed.

So `program.md` is best understood not as documentation, but as an interface. More precisely, it is an LLM-native control plane for the research loop.

## Code-Level Analysis 4: `analysis.ipynb`

This notebook gets less attention, but it is part of the design.

It reads `results.tsv` and then:

- extracts kept experiments
- visualizes how the frontier moves downward over time
- computes total improvement over baseline
- ranks the most important improvements

That tells us something important. The project is not trying to automate everything. The intended division of labor looks more like this:

- the agent performs repeated local search
- the human compresses, interprets, and generalizes what happened

So the system is not fully autonomous science. It is automated exploration with human synthesis layered on top.

## How `autoresearch` Relates to `nanochat`

`autoresearch` is not a replacement for `nanochat`. It is a distilled form of `nanochat`'s pretraining research surface, collapsed into a single-GPU, single-file, agent-editable harness.

The comparison is revealing:

| Dimension | `nanochat` | `autoresearch` |
|---|---|---|
| Purpose | end-to-end LLM harness | autonomous pretraining research loop |
| Compute | CPU, MPS, CUDA, includes DDP | primarily single NVIDIA GPU |
| Editable surface | many modules plus CLI | effectively one file: `train.py` |
| Evaluation | `val_bpb`, CORE, sampling, reporting | one dominant metric: `val_bpb` |
| Operations | wandb, checkpoints, resume, distributed execution | `run.log` plus `results.tsv` |
| Human role | runs and interprets training | designs `program.md` and interprets outcomes |
| Agent role | optional assistant | main experimental operator |

### What `autoresearch` keeps from `nanochat`

- the GPT trunk design
- value embeddings and gate logic
- the Muon plus AdamW optimization philosophy
- BOS-aligned best-fit dataloading
- `val_bpb`-centered evaluation
- meta-device initialization and `torch.compile`-based optimization

### What `autoresearch` removes

- DDP and distributed training
- checkpoint and resume complexity
- wandb logging
- CORE evaluation
- chat, SFT, RL, and the web UI
- broader multi-platform fallback logic
- a wide CLI configuration surface

The key point is simple: `autoresearch` is not just removing features for minimalism. It is removing the dimensions that are harder for agents to operate reliably.

## What Round 1 Actually Found

This is the most interesting part. Commit `6ed7d1d` in `nanochat` explicitly summarizes the changes attributed to `autoresearch` round 1.

To make that less abstract, here is a stylized experiment loop. This is reconstructed from the **public** `program.md` protocol and the round 1 changes that were eventually merged. It is not meant as a literal replay of the exact internal loop used in every round 1 experiment.

```diff
- UNEMBEDDING_LR = 0.004
+ UNEMBEDDING_LR = 0.008
```

1. The agent runs the baseline and records the reference `val_bpb`.
2. It changes one or a few lines near the top of `train.py`.
3. It runs `uv run train.py > run.log 2>&1`.
4. It reads `val_bpb` and `peak_vram_mb` out of the log.
5. If the result is better, it keeps the commit and logs it in `results.tsv`.
6. If the result is worse, it resets and moves on.

The point is not that any one patch is glamorous. The point is that these small diffs can accumulate overnight under agent control.

### Optimization and scheduling changes

- unembedding LR: `0.004 -> 0.008`
- weight decay: `0.2 -> 0.28`
- Adam betas and weight decay split by parameter group
- Muon `beta2`: `0.95 -> 0.9`
- Muon momentum warmup target: `0.95 -> 0.97` over 400 steps
- warmup changed from a ratio to an absolute 40-step schedule
- warmdown ratio: `0.5 -> 0.65`
- final LR fraction: `0.0 -> 0.05`
- weight decay schedule: linear decay to cosine decay
- polar express normalization factor: `1.02 -> 1.01`

### Architecture and initialization changes

- VE gate channels: `32 -> 12`
- gate scale range: `2x -> 3x`
- gate init: zero to small positive
- post-QK-norm scaling added: `q, k *= 1.15`
- embedding init std: `1.0 -> 0.8`
- MLP `c_fc` init scaled down by 0.5x
- RoPE base theta: `10K -> 100K`
- short attention window: from `seq_len / 2` to roughly `seq_len / 3`
- logit softcap: `20 -> 15`

These changes naturally split into two categories.

### 1. Changes that improve sample efficiency

- larger unembedding LR
- parameter-group-specific optimizer settings
- Muon beta2 and momentum tuning
- sharper attention through QK scaling
- initialization changes
- logit softcap tuning

These are about learning better from the same number of tokens.

### 2. Changes that improve throughput or efficiency under the 5-minute budget

- shrinking the short attention window
- reshaping warmup and warmdown
- changing time-to-learning behavior through the schedule

This is where `autoresearch` becomes especially interesting. The move from a half-context short window to roughly a third-context short window is not just a model-quality decision. It reflects the actual objective: not the best model in the abstract, but the best result under a fixed 300-second budget.

That is the signature of the whole project. Model research and systems research are fused together.

## Why This Design Is Strong

### 1. The search space is narrow enough

The agent mostly edits `train.py`. That keeps diffs small, failures easier to debug, and reverts cheap.

### 2. The evaluation function is clear enough

Everything is compared through `val_bpb`. This is not a blurry multi-metric objective.

### 3. The time budget aligns research with actual compute

Because the objective is fixed-time rather than fixed-step, the agent explores model quality and execution efficiency together.

### 4. Git serves as memory

Good experiments remain as commits. Bad experiments get reset. `results.tsv` becomes the experiment ledger.

### 5. Natural language becomes the orchestration layer

Without a giant custom framework, `program.md` can already define a surprisingly capable research protocol.

## Limits and Critical Caveats

`autoresearch` is impressive, but it should not be romanticized. It is a very smart local-search engine, not a scientific discovery machine.

### 1. The dangers of single-metric optimization

The keep-or-discard decision is based on `val_bpb`. That is efficient, but it can miss other forms of capability or generalization.

### 2. Pinned-validation bias

Fixing validation to one shard makes comparison easier, but it leaves open the possibility of optimizing toward that shard's specific characteristics.

### 3. Single-run decision-making

There is no statistical replication inside the main loop. A single 5-minute run is enough to keep or discard an idea. In noisy settings, that can produce false positives or false negatives.

### 4. A hardware-specific objective

As the README itself notes, `autoresearch` finds the best model for your platform under that budget. Different GPUs, kernels, and memory constraints can change what "best" means.

### 5. Portability is limited

The public baseline is strongly CUDA- and NVIDIA-oriented. `train.py` directly uses `torch.device("cuda")` and ties multiple decisions to CUDA execution paths.

### 6. The repo is small, but the stack is still sharp-edged

`torch.compile`, fused optimizer logic, external kernel loading, and CUDA capability branching are all useful for speed, but they also make the system more brittle if the agent changes too much at once.

### 7. This is not a fully autonomous lab

Humans still do critical work:

- define and refine `program.md`
- choose the protocol
- interpret the results
- decide which findings deserve promotion back into upstream code

So this is not human-out-of-the-loop. It is closer to human-above-the-loop.

## Why It Still Matters Anyway

The deeper significance of `autoresearch` is not "AI replaced the researcher." It is that research work is beginning to be repackaged into a form that AI can productively execute.

More concretely:

- old pattern: humans change code, run models, read logs, and iterate
- `autoresearch` pattern: humans write the operating protocol, and agents execute the code-change and experiment loop

That shifts the human role away from implementation and toward protocol design.

That shift may be a bigger deal than it first appears. The advantage may increasingly go not to whoever writes code fastest by hand, but to whoever designs the best autonomous search environment.

## My Read: The Real Output Is a Research Operating System

On the surface, the output of `autoresearch` is a better `val_bpb`, or a faster `nanochat`. But structurally, I think the more important output is a research operating system.

Its components are:

- a fixed experiment constitution: `prepare.py`
- an editable search surface: `train.py`
- a natural-language protocol: `program.md`
- experiment memory: git plus `results.tsv`
- post-hoc interpretation tools: `analysis.ipynb`

That pattern is likely reusable well beyond LLM pretraining. The same structure can apply anywhere you have:

- one dominant metric
- one fixed budget
- a deliberately limited editable surface

That could include inference latency tuning, compiler flag search, prompt-eval optimization, retrieval pipeline tuning, and similar problems.

## How It Differs From HPO, AutoML, and NAS

At a glance, `autoresearch` is obviously an automated search system. But it differs in an important way from older automation categories.

| Approach | What it usually searches | How `autoresearch` differs |
|---|---|---|
| HPO | numbers on top of fixed code, like LR or batch size | `autoresearch` edits the code in `train.py` itself, while still keeping the editable surface narrow enough to remain stable |
| NAS | architectural structures and connectivity | `autoresearch` searches architecture, optimizer, initialization, schedule, and execution behavior at the same time |
| AutoML | model selection, preprocessing, and pipeline automation | `autoresearch` automates the lab SOP itself through git, `program.md`, and keep-discard rules rather than through a separate heavyweight controller |

Seen this way, the project is less a search script and more a lightweight research OS.

## What Practitioners Should Learn From It

The practical lessons are clear.

### 1. Do not just make the agent smarter. Make the problem more agent-friendly.

Restrict the editable surface, fix the metric, and limit the runtime budget. Agent performance improves dramatically when the task is shaped correctly.

### 2. Natural-language documents can become execution layers

`program.md` is not just explanatory text. It is an orchestration layer. In AI-native systems, documents may increasingly become operational interfaces.

### 3. Small autonomous loops win before giant ones do

A 5-minute automated research loop can create value sooner than an attempt to build a fully autonomous company.

### 4. Metric design may matter more than agent design

Even a very capable agent will drift if the metric is fuzzy. `autoresearch` shows how much clarity comes from a sharp objective.

## Conclusion

`autoresearch` is not the final form of automated research. But the direction is unusually clear.

The repo does not hand the agent unlimited freedom. It gives the agent a constrained surface and a precise objective. Under those conditions, the agent appears capable of finding useful improvements that can be promoted back into a real upstream codebase.

To me, the most important message is not "AI can do research." It is:

Research can be redefined into an interface that AI can productively execute.

And the key ingredients are surprisingly small:

- one fixed evaluation function
- one editable training file
- one natural-language operating protocol
- one keep-or-discard loop

At that point, the lab is already halfway to being software.

## References

- `autoresearch` README: [https://github.com/karpathy/autoresearch/blob/master/README.md](https://github.com/karpathy/autoresearch/blob/master/README.md)
- `autoresearch` `program.md`: [https://github.com/karpathy/autoresearch/blob/master/program.md](https://github.com/karpathy/autoresearch/blob/master/program.md)
- `autoresearch` `prepare.py`: [https://github.com/karpathy/autoresearch/blob/master/prepare.py](https://github.com/karpathy/autoresearch/blob/master/prepare.py)
- `autoresearch` `train.py`: [https://github.com/karpathy/autoresearch/blob/master/train.py](https://github.com/karpathy/autoresearch/blob/master/train.py)
- `nanochat` README: [https://github.com/karpathy/nanochat/blob/master/README.md](https://github.com/karpathy/nanochat/blob/master/README.md)
- `nanochat` round 1 integration commit: [https://github.com/karpathy/nanochat/commit/6ed7d1d82cee16c2e26f45d559ad3338447a6c1b](https://github.com/karpathy/nanochat/commit/6ed7d1d82cee16c2e26f45d559ad3338447a6c1b)
- Leaderboard update commit: [https://github.com/karpathy/nanochat/commit/f06860494848db080c9a80a0ffa83203b042056b](https://github.com/karpathy/nanochat/commit/f06860494848db080c9a80a0ffa83203b042056b)
- X post: [https://x.com/karpathy/status/2031135152349524125](https://x.com/karpathy/status/2031135152349524125)
- Text mirror: [https://xcancel.com/karpathy/status/2031135152349524125](https://xcancel.com/karpathy/status/2031135152349524125)
