---
name: experiment-autotune-loop
description: "24-hour autonomous experiment loop: monitor running jobs, let Codex analyze failures and results, tune hyperparameters, relaunch experiments, and repeat until the target metric is reached or timeout. Use when user says \"24小时监控\", \"自动调参\", \"重跑实验\", \"调参循环\", \"直到达标\", or wants Codex to keep iterating on experiments."
argument-hint: [experiment-scope-or-launch-command]
---

# Experiment Autotune Loop

Autonomously run a 24-hour loop:

`launch -> monitor -> diagnose -> tune -> relaunch -> compare -> stop at target or timeout`

Use this after the experiment is already runnable. If the code or launch command does not exist yet, use `/experiment-bridge` or `/run-experiment` first.

## Context

$ARGUMENTS

## Constants

- `TIMEOUT = 24h`
- `PROCESS_CHECK_INTERVAL = 60s`
- `RESULT_CHECK_INTERVAL = 15min`
- `MAX_TUNE_ROUNDS = 12`
- `PATIENCE = 4` consecutive non-improving completed runs
- `MAX_RESTARTS_PER_CONFIG = 2`
- `MAX_NEW_CONFIGS_PER_ROUND = 2`
- `TARGET_DIRECTION = maximize`
- `LOOP_DIR = outputs/experiment_autotune_loop`
- `REVIEWER_MODEL = gpt-5.4`

Override inline:

```text
/experiment-autotune-loop "python train.py --config configs/base.yaml --seed 1 -- metric: eval/return, target: >= 82, direction: maximize, timeout: 18h, patience: 3"
```

## Required Inputs

Before starting the loop, confirm or infer:

1. A runnable launch command or command template
2. A primary metric, direction (`maximize` or `minimize`), and success threshold
3. Where the metric comes from: JSON, CSV, WandB, stdout log, or evaluation script
4. The tunable parameters and their current defaults
5. The runtime environment from project notes (`AGENTS.md`, `CLAUDE.md`, scripts, configs)

If the success threshold is not explicitly given, infer it from `EXPERIMENT_PLAN.md`, tracker files, or project notes. Ask the user only if it cannot be recovered safely.

## Loop Files

Create `LOOP_DIR/` with:

- `LOOP_STATE.json` - current round, best metric, pending run, timeout state
- `RUN_HISTORY.csv` - one row per launch attempt
- `LOOP_LOG.md` - narrative log of decisions and outcomes
- `best_config.json` - current best-known config and launch command

Write state after every completed diagnosis or launch so the loop can recover after compaction or interruption.

## State Recovery

When the skill is invoked:

1. If `LOOP_STATE.json` does not exist: fresh start
2. If `status = completed`: fresh start unless the user asks to continue from the old best config
3. If `status = in_progress` and the timestamp is within `TIMEOUT`: resume
4. If `status = in_progress` but stale beyond `TIMEOUT`: mark stale in `LOOP_LOG.md`, then start fresh

On resume, read:

- `LOOP_STATE.json`
- `RUN_HISTORY.csv`
- `LOOP_LOG.md`

Then continue from the next pending check instead of restarting the whole search.

## Workflow

### Phase 0: Parse and Prepare

1. Read the project to locate:
   - training and evaluation commands
   - config files or argparse flags
   - result files and log paths
   - known baselines and expected runtime
2. If no parser exists for the target metric, write a small parser before launching the loop. Do not rely on eyeballing logs.
3. Identify tunable parameters. Start with a conservative set:
   - learning rate
   - batch size / gradient accumulation
   - warmup / scheduler
   - regularization
   - rollout horizon / update ratio / penalty coefficients for RL-style code
4. Record the baseline config in `best_config.json`.
5. If the job runs remotely, read `docs/WATCHDOG_GUIDE.md` or `docs/WATCHDOG_GUIDE_CN.md` and set up `tools/watchdog.py` for continuous process health checks.

### Phase 1: Launch Baseline or Resume Best Pending Run

If there is no valid completed run in `RUN_HISTORY.csv`, launch the baseline first.

Use `/run-experiment` semantics:

- pick an available GPU
- save stdout/stderr with `tee`
- keep a stable session name
- record the exact command in `RUN_HISTORY.csv`

After launch, register the run with `tools/watchdog.py` when watchdog is available.

### Phase 2: Monitor

Use two layers of monitoring together:

1. **Process health** every `PROCESS_CHECK_INTERVAL`
   - watchdog status
   - session alive or dead
   - GPU active or idle
2. **Training and result quality** every `RESULT_CHECK_INTERVAL`
   - `/monitor-experiment` for logs, result files, checkpoints, and completion
   - `/training-check` for loss trend, NaN, divergence, plateau, or metric degradation when WandB or structured logs exist

Persist the latest observation in `LOOP_STATE.json` after each deep check.

### Phase 3: Classify What Happened

Classify the latest run into exactly one bucket:

| Bucket | Meaning | Next action |
|--------|---------|-------------|
| `SUCCESS` | Target reached | Stop loop and write final report |
| `PROMISING` | Better than best but below target | Save as best, optionally confirm with another seed |
| `NO_IMPROVEMENT` | Finished normally but did not beat best | Tune next config |
| `QUALITY_FAILURE` | NaN, divergence, severe overfit, no learning | Tune for stability or training quality |
| `INFRA_FAILURE` | session dead, idle GPU too early, OOM, disk/network issue | Fix infra and restart same config first |
| `AMBIGUOUS` | signal unclear | trigger bounded Codex review or do local diagnosis |

Infrastructure failures do **not** count as tuning rounds unless the config itself caused the failure repeatedly.

### Phase 4: Codex Diagnosis and Tuning

For every non-success case, write a short diagnosis entry to `LOOP_LOG.md`:

- observed symptoms
- likely cause
- next action
- expected metric impact

Tune with small, auditable moves. Default rule:

- change at most `1` major knob and `1-2` minor knobs per round
- never change optimizer, model size, data pipeline, and schedule all at once
- keep the previous best config as the parent of every new config

#### Common symptom -> action mapping

| Symptom | Likely cause | First action |
|--------|--------------|--------------|
| OOM / kill | memory pressure | reduce batch, add grad accumulation, enable mixed precision |
| NaN / exploding loss | update too aggressive | lower learning rate, add gradient clipping, increase warmup |
| Flat metric, loss also flat | under-training or weak signal | modestly raise learning rate, train longer, reduce regularization |
| Train improves but eval worsens | overfitting | stronger regularization, earlier stopping, more augmentation |
| Large variance across seeds | unstable conclusion | rerun best config with another seed before widening search |
| Best run sits at search boundary | range too narrow | extend that parameter one step further |

If there have been at least 3 completed runs, use `/analyze-results` or an equivalent local comparison to summarize which parameters correlate with improvement. If unavailable, do the comparison locally.

### Optional Secondary Codex Review

Only for ambiguous cases or after two weak tuning rounds in a row:

```text
Need one bounded tuning judgment.

Task: pick the next experiment action for this run loop.
Current target: [metric, threshold, direction]
Best completed run: [config + metric]
Last 3 runs:
- [config diff] -> [metric / failure]
- [config diff] -> [metric / failure]
- [config diff] -> [metric / failure]

Respond with exactly one:
- RESTART_SAME
- CONFIRM_BEST_WITH_NEW_SEED
- TUNE_NEXT: <1 major knob + up to 2 minor knobs>
- STOP: diminishing returns
```

If delegation is unavailable, make the same decision locally and mark it `[local judgment]`.

### Phase 5: Relaunch

For the chosen next action:

1. Create the new config or command diff
2. Record the parent config and rationale
3. Launch via the same mechanism as the baseline
4. Register the run with watchdog if used
5. Append a row to `RUN_HISTORY.csv`:

```csv
round,run_name,parent_run,status,metric,decision,config_diff,notes,timestamp
```

When the decision is `RESTART_SAME`, increment the restart counter for that config. If it exceeds `MAX_RESTARTS_PER_CONFIG`, escalate from infra retry to real tuning or stop.

### Phase 6: Stop Conditions

Stop immediately when any one is true:

1. Target metric reached
2. Wall-clock timeout reached
3. `MAX_TUNE_ROUNDS` exhausted
4. `PATIENCE` consecutive completed runs without improvement
5. Same failure repeats three times across distinct configs
6. Codex concludes the next moves are dominated by compute cost or risk

## Final Report

On termination, write `LOOP_LOG.md` summary:

```markdown
# Experiment Autotune Loop Report

- Target: [metric threshold]
- Timeout budget: [X]
- Total runs launched: [N]
- Best metric: [value]
- Best config: [path or command]
- Stopping reason: [success / timeout / patience / repeated failure]

## Best Run
- Command:
- Config diff vs baseline:
- Evidence:

## Trajectory
- Round 0: baseline -> ...
- Round 1: ...

## Recommendation
- [rerun more seeds / scale up / stop here / hand off to paper workflow]
```

Also set `LOOP_STATE.json` to `"status": "completed"`.

## Key Rules

- Use watchdog for continuous health, not for metric judgment.
- Use `/training-check` for training quality, not for final result comparison.
- Always save raw numbers before interpreting them.
- Prefer cheap fixes before expensive ones.
- Confirm promising gains with another seed before making big search jumps.
- Keep failed runs in `RUN_HISTORY.csv`; failed evidence is part of the search trace.
- If a run is still healthy and improving, do not interrupt it just because a different config looks tempting.
- If the project already has an experiment tracker, append to it instead of creating a competing source of truth.

## Composing With Other Skills

- `/run-experiment` - launch or relaunch jobs
- `/monitor-experiment` - inspect logs and collect outputs
- `/training-check` - catch low-quality training early
- `/analyze-results` - compare finished runs and identify trends
- `/result-to-claim` - convert the final best results into paper-safe claims
