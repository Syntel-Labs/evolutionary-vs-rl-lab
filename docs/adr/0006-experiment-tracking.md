# ADR 0006 — Experiment Tracking

## Status

Accepted

---

## Context

It was necessary to decide how to organize, identify, and manage the complete set of experiments in the project, ensuring that each run is uniquely identifiable, that results are not overwritten between executions, and that the final comparative analysis can be automatically aggregated from the generated artifacts.

The problem is different from logging (ADR 0005). While ADR 0005 defines what is recorded within a run, this ADR defines how the set of all runs is organized as a complete experiment.

Active constraints at the time of the decision:

* 4 methods × 10 seeds = 40 runs in the main experiment
* Robustness tests generate additional runs
* No external tracking server
* Results must be automatically aggregable by script
* Structure must survive partial re-execution of experiments without corrupting existing results

---

## Options Considered

### Option 1 — MLflow Tracking Server

Advantages: full UI, run search, automatic comparison, centralized storage.

Discarded because it requires a local or remote server, adds infrastructure complexity, and creates a service dependency that makes reproducibility harder in other environments.

### Option 2 — Weights & Biases Projects

Advantages: automatic run organization, visual comparison, tags and groups.

Discarded for the same reasons as in ADR 0005: dependency on an external service and required internet connection.

### Option 3 — Manual run numbering

Advantages: extreme simplicity.

Discarded because it does not scale to 40 runs, is prone to human error, and does not allow automatic identification of runs by attributes such as method or seed.

### Option 4 — Hierarchical directory structure + central registry

Advantages: no external dependencies, predictable structure, script-aggregable, survives partial executions, unique identification by path.

**Selected.**

---

## Decision

A hierarchical directory structure is used as the tracking system, complemented by a central JSON registry that aggregates the state of all runs.

### Directory structure

```bash
runs/
  experiment_main/
    ga/
      seed_11/
      seed_23/
      ...
      seed_101/
    dqn/
      seed_11/
      ...
    ppo/
      seed_11/
      ...
    random/
      seed_11/
      ...
  experiment_robustness/
    ga/
      seed_11_gravity_high/
      seed_11_gap_low/
      seed_11_speed_high/
      ...
    dqn/
      ...
    ppo/
      ...
```

### Unique run identification

Each run is identified by its full path:

$$\texttt{experiment/method/seed_{s}}$$

This identifier is sufficient to locate all artifacts of a run unambiguously.

### Central registry

Upon completion of each run, a central file is updated:

```bash
runs/
  registry.json
```

With the following structure per entry:

```json
{
  "experiment_main/dqn/seed_11": {
    "status": "completed",
    "method": "dqn",
    "seed": 11,
    "env_seed": 11,
    "agent_seed": 1011,
    "total_steps": 2000000,
    "wall_time_seconds": 0,
    "converged": true,
    "r_final": 0.0,
    "steps_to_threshold": 0,
    "auc": 0.0,
    "timestamp_start": "",
    "timestamp_end": "",
    "artifacts": [
      "config.json",
      "logs.csv",
      "metrics.json"
    ]
  }
}
```

### Possible run states

| State       | Description                                   |
| ----------- | --------------------------------------------- |
| `pending`   | Registered but not started                    |
| `running`   | Currently executing                           |
| `completed` | Finished within budget                        |
| `failed`    | Terminated due to a technical error           |
| `excluded`  | Excluded from analysis with documented reason |

### Aggregation script

A script `aggregate.py` reads `registry.json` and all `metrics.json` files to build the final comparative table:

$$\text{table}: \text{method} \times \text{seed} \to
{R_{\text{final}},; S_T,; \text{AUC},; \text{CV},;
\text{converged}}$$

This script is the direct input to the statistical analysis defined in `06_metrics_definition.md`.

### Overwrite protection

Before starting a run, the runner verifies:

```bash
if path exists and status == "completed":
    skip
```

This allows re-running the entire experiment without overwriting already completed runs, executing only pending or failed ones.

---

## Consequences

### Positive

* No external dependencies: works in any environment
* Self-documented and auditable directory structure
* Unique identification by path eliminates collisions
* Overwrite protection enables incremental execution
* `registry.json` provides a global view of experiment state
* `aggregate.py` automates construction of the final comparative table
* Compatible with version control: paths are deterministic

### Negative

* No visualization UI: experiment status is only visible by reading `registry.json` or running the script
* `registry.json` may have conflicts if two runs execute in parallel and write simultaneously
* No attribute-based search like in MLflow or wandb

---

### Impact on other components

The directory structure defined here is what ADR 0005 assumes to locate `logs.csv`, `metrics.json`, and checkpoints. Both ADRs must remain consistent.

The `aggregate.py` script is the bridge between logging artifacts (ADR 0005) and the statistical analysis defined in `06_metrics_definition.md`. Any change in the fields of `metrics.json` must be reflected in this script.

The separation between `experiment_main` and `experiment_robustness` in the directory structure ensures that the robustness tests defined in `05_experimental_design.md` do not contaminate the main metrics of the comparative experiment.
