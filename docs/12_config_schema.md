# 12 — Config Schema

This document defines the formal structure of all configuration files in the project. Any parameter that affects experiment behavior must be in a configuration file, never hardcoded.

The files live in `configs/`. Each run generates a `config.json` snapshot in its directory that freezes the exact configuration used (see ADR 0006).

---

## File Structure

```bash
configs/
├── env.yaml            — environment parameters
├── ga.yaml             — GA hyperparameters
├── dqn.yaml            — DQN hyperparameters
├── ppo.yaml            — PPO hyperparameters
└── experiment.yaml     — seeds, budget, frequencies
```

---

## env.yaml

Environment parameters defined in `03_environment_spec.md`.

```yaml
env:
  # Physical dynamics
  gravity: 0.5
  jump_impulse: -8.0
  pipe_speed: 3.0

  # Geometry
  pipe_gap: 120
  pipe_spacing: 250
  screen_width: 400
  screen_height: 500
  y_min: 0
  y_max: 500

  # Episode
  max_steps: 10000

  # Reproducibility
  seed: null                  # overridden by experiment.yaml

  # Normalization
  v_max: 15.0                 # clip for velocity normalization

  # Optional
  randomize_start: false      # randomize initial position
  action_repeat: 1            # frameskip (1 = no frameskip)
  render_mode: null           # null = no render, "human" = 30fps
```

### Types and constraints

| Parameter       | Type  | Constraint                  |
| --------------- | ----- | --------------------------- |
| `gravity`       | float | $> 0$                       |
| `jump_impulse`  | float | $< 0$                       |
| `pipe_speed`    | float | $> 0$                       |
| `pipe_gap`      | int   | $> 0$, $< $ `screen_height` |
| `pipe_spacing`  | int   | $> $ `screen_width`         |
| `max_steps`     | int   | $> 0$                       |
| `v_max`         | float | $> 0$                       |
| `action_repeat` | int   | $\geq 1$                    |

---

## ga.yaml

GA hyperparameters defined in `04_methodology/genetic_algorithm.md`.

```yaml
ga:
  # Population
  population_size: 100
  elite_fraction: 0.05        # E = floor(elite_fraction × population_size)

  # Initialization
  init_std: 0.1               # σ_init ~ N(0, init_std²)

  # Selection
  tournament_size: 3          # k in binary tournament

  # Crossover
  crossover_prob: 0.8         # p_c
  crossover_type: "uniform"   # uniform | arithmetic | one_point

  # Mutation
  mutation_prob: null         # null = 1/d (standard), or explicit float
  mutation_std: 0.05          # σ_mut

  # Budget
  # Inherited from experiment.yaml → budget_steps
```

### Types and constraints

| Parameter         | Type       | Constraint                     |
| ----------------- | ---------- | ------------------------------ |
| `population_size` | int        | $> 0$                          |
| `elite_fraction`  | float      | $\in (0, 1)$                   |
| `init_std`        | float      | $> 0$                          |
| `tournament_size` | int        | $\geq 2$                       |
| `crossover_prob`  | float      | $\in [0, 1]$                   |
| `crossover_type`  | str        | uniform, arithmetic, one_point |
| `mutation_prob`   | float\|null | $\in (0, 1)$ or null           |
| `mutation_std`    | float      | $> 0$                          |

---

## dqn.yaml

DQN hyperparameters defined in `04_methodology/dqn.md`.

```yaml
dqn:
  # Neural network
  # Architecture fixed by ADR 0002, not configurable here
  # Enforced via policy_kwargs in SB3

  # Learning
  learning_rate: 0.0001
  gamma: 0.99
  batch_size: 32
  gradient_clip: 10.0

  # Replay buffer
  buffer_size: 100000
  warmup_steps: 1000

  # Target network
  target_update_interval: 1000   # τ_update in steps

  # Update frequency
  train_freq: 4                  # Δ_update in steps

  # Exploration
  epsilon_start: 1.0
  epsilon_end: 0.05
  epsilon_decay_steps: 500000

  # Implementation
  policy: "MlpPolicy"
  verbose: 0
```

### Types and constraints

| Parameter                | Type  | Constraint                         |
| ------------------------ | ----- | ---------------------------------- |
| `learning_rate`          | float | $> 0$                              |
| `gamma`                  | float | $\in (0, 1]$                       |
| `batch_size`             | int   | $> 0$                              |
| `gradient_clip`          | float | $> 0$                              |
| `buffer_size`            | int   | $> $ `batch_size`                  |
| `warmup_steps`           | int   | $\geq 0$                           |
| `target_update_interval` | int   | $> 0$                              |
| `train_freq`             | int   | $\geq 1$                           |
| `epsilon_start`          | float | $\in (0, 1]$                       |
| `epsilon_end`            | float | $\in (0, 1)$, $< $ `epsilon_start` |
| `epsilon_decay_steps`    | int   | $> 0$                              |

---

## ppo.yaml

PPO hyperparameters. Exact values are defined in `04_methodology/ppo.md` (pending). The fields here define the expected schema.

```yaml
ppo:
  # Neural network
  # Architecture fixed by ADR 0002, not configurable here

  # Learning
  learning_rate: 0.0003
  gamma: 0.99
  gae_lambda: 0.95
  clip_range: 0.2
  ent_coef: 0.01
  vf_coef: 0.5
  max_grad_norm: 0.5

  # Rollout
  n_steps: 2048               # steps per rollout per environment
  batch_size: 64
  n_epochs: 10                # epochs per rollout

  # Implementation
  policy: "MlpPolicy"
  verbose: 0
```

### Types and constraints

| Parameter       | Type  | Constraint        |
| --------------- | ----- | ----------------- |
| `learning_rate` | float | $> 0$             |
| `gamma`         | float | $\in (0, 1]$      |
| `gae_lambda`    | float | $\in (0, 1]$      |
| `clip_range`    | float | $\in (0, 1)$      |
| `ent_coef`      | float | $\geq 0$          |
| `vf_coef`       | float | $> 0$             |
| `max_grad_norm` | float | $> 0$             |
| `n_steps`       | int   | $> 0$             |
| `batch_size`    | int   | $\leq $ `n_steps` |
| `n_epochs`      | int   | $\geq 1$          |

---

## experiment.yaml

Experiment parameters defined in `05_experimental_design.md`.

```yaml
experiment:
  # Identification
  name: "experiment_main"     # experiment_main | experiment_robustness

  # Budget
  budget_steps: 2000000

  # Seeds
  seeds: [11, 23, 37, 41, 53, 67, 79, 83, 97, 101]
  agent_seed_offset: 1000     # agent_seed = seed + offset
  eval_seed: 999

  # Evaluation
  eval_freq: 50000            # steps between intermediate evaluations
  eval_episodes: 10           # episodes per intermediate evaluation
  final_eval_episodes: 50     # episodes in final evaluation

  # Logging
  log_freq: 10000             # steps between logs.csv entries
  checkpoint_freq: 250000     # steps between checkpoints

  # Execution order
  methods: ["ga", "dqn", "ppo", "random"]

  # Robustness tests (only for experiment_robustness)
  robustness:
    gravity_factor: 1.1
    gap_factor: 0.9
    speed_factor: 1.1
```

### Types and constraints

| Parameter             | Type      | Constraint                             |
| --------------------- | --------- | -------------------------------------- |
| `name`                | str       | experiment_main, experiment_robustness |
| `budget_steps`        | int       | $> 0$                                  |
| `seeds`               | list[int] | length $\geq 1$, no duplicates         |
| `agent_seed_offset`   | int       | $> 0$                                  |
| `eval_seed`           | int       | $\notin $ `seeds`                      |
| `eval_freq`           | int       | divisor of `budget_steps`              |
| `eval_episodes`       | int       | $> 0$                                  |
| `final_eval_episodes` | int       | $\geq $ `eval_episodes`                |
| `log_freq`            | int       | $\leq $ `eval_freq`                    |
| `checkpoint_freq`     | int       | multiple of `eval_freq`                |
| `methods`             | list[str] | subset of [ga, dqn, ppo, random]       |

---

## Config Snapshot

At the start of each run, the runner serializes the full effective configuration into `runs/{experiment}/{method}/seed_{s}/config.json`:

```json
{
  "experiment": { },
  "env": { },
  "method": "dqn",
  "method_config": { },
  "seed": 11,
  "env_seed": 11,
  "agent_seed": 1011,
  "timestamp_start": ""
}
```

This snapshot is immutable once generated. It allows exact reproduction of any run from its own artifacts, regardless of subsequent changes in `configs/`.

---

## General Rules

* Any parameter that affects results must be in a yaml file, never hardcoded in code
* The defaults in `env.yaml` correspond to the values in `03_environment_spec.md`
* `gamma = 0.99` is consistent across `dqn.yaml`, `ppo.yaml`, and `02_problem_definition.md`
* Any change in a yaml after experiments have started invalidates previous runs unless the config snapshot confirms the value was different
* Fields marked as inherited are not duplicated: `budget_steps` exists only in `experiment.yaml`
