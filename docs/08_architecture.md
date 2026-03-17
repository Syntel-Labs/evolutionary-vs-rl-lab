# 08 — Architecture

## Overview

The project is organized as a modular experimental system with strict separation between environment, agents, models, experiments, and analysis. No component knows the internal details of another: all communicate through defined interfaces.

```bash
evolution-vs-learning/
├── env/
├── agents/
│   ├── random/
│   ├── genetic/
│   ├── dqn/
│   └── ppo/
├── models/
├── experiments/
├── logging/
├── runs/
├── analysis/
├── configs/
└── scripts/
```

---

## Main Data Flow

```bash
config.yaml
    │
    ▼
ExperimentRunner
    │
    ├──► Env (Gymnasium API)
    │         │
    │    observation, reward,
    │    terminated, truncated
    │         │
    ├──► Agent (GA / DQN / PPO / Random)
    │         │
    │       action
    │         │
    │    [updates parameters internally]
    │         │
    └──► Logger
              │
         logs.csv
         metrics.json
         checkpoints/
         registry.json
```

The environment does not know the agent. The agent does not know the logger. The runner orchestrates the interaction between the three.

---

## Modules

### `env/`

Contains the Gymnasium-compatible Flappy Bird environment.

```bash
env/
├── flappy_env.py       — main class, implements Gymnasium API
├── physics.py          — separated physics dynamics
├── obstacle.py         — obstacle generation and movement
└── wrappers.py         — observation normalization
```

Responsibilities:

* Implement `reset()`, `step()`, `render()`
* Apply input normalization defined in `03_environment_spec.md`
* Control `env_seed` independently of the agent
* Return `(observation, reward, terminated, truncated, info)`

Does not know the agent. Does not know the logger.

### `models/`

Contains exclusively network architecture definitions.

```bash
models/
└── mlp.py      — 2-layer MLP, 64 neurons, ReLU (ADR 0002)
```

Responsibilities:

* Define the shared base architecture for GA, DQN, and PPO
* Expose number of parameters $d$ so GA can build the chromosome
* No training or optimization logic

Does not know the agent. Does not know the environment.

### `agents/`

Each agent implements a common interface:

```python
class BaseAgent:
    def select_action(self, observation) -> int
    def update(self, transition) -> None
    def get_weights(self) -> np.ndarray
    def set_weights(self, weights: np.ndarray) -> None
    def save(self, path: str) -> None
    def load(self, path: str) -> None
```

```bash
agents/
├── base.py             — BaseAgent interface
├── random/
│   └── random_agent.py
├── genetic/
│   ├── ga_agent.py     — orchestrates evolution
│   ├── population.py   — population management
│   ├── operators.py    — crossover, mutation, selection
│   └── chromosome.py   — vector ↔ network mapping (ADR 0003)
├── dqn/
│   └── dqn_agent.py    — SB3 DQN wrapper (ADR 0004)
└── ppo/
    └── ppo_agent.py    — SB3 PPO wrapper
```

Responsibilities by agent:

**Random**: selects $a_t \sim \text{Uniform}({0,1})$. No internal state. No update.

**GA**: maintains a population of chromosomes. In each generation it evaluates individuals in the environment, applies selection, crossover, and mutation, and updates the population. It does not call `update()` per transition but per full episode.

**DQN**: wrapper over SB3 with architecture enforced via `policy_kwargs`. Exposes the same interface as the other agents.

**PPO**: wrapper over SB3 with the same base architecture. Exposes the same interface.

### `logging/`

Contains the logging system defined in ADR 0005.

```bash
logging/
├── logger.py       — writes logs.csv and metrics.json
├── csv_writer.py   — structured step-level writing
└── tb_writer.py    — TensorBoard adapter for GA
```

Responsibilities:

* Write `logs.csv` every $10{,}000$ steps
* Write `metrics.json` at the end of each run
* Write TensorBoard events compatible with SB3
* Register entry in `registry.json` upon completion

Does not know the environment. Does not know the agent. Receives data from the runner.

### `experiments/`

Contains the runner and orchestration logic.

```bash
experiments/
├── runner.py           — ExperimentRunner, orchestrates full run
├── evaluator.py        — separate evaluation phase
└── robustness.py       — post-training perturbation tests
```

`ExperimentRunner` is the only component that knows the environment, the agent, and the logger simultaneously. Its responsibility is:

1. Read config
2. Instantiate environment, agent, and logger
3. Execute training loop up to $2\text{M}$ steps
4. Call intermediate evaluation every $50{,}000$ steps
5. Call final evaluation at the end of the budget
6. Write entry to `registry.json`

```bash
loop:
    obs = env.reset()
    while steps < budget:
        action = agent.select_action(obs)
        obs, reward, terminated, truncated, info = env.step(action)
        agent.update(transition)
        logger.log(step, reward, ...)
        if steps % 50_000 == 0:
            evaluator.run(agent, env)
        if terminated or truncated:
            obs = env.reset()
```

For GA, the internal loop differs: instead of `update()` per transition, the runner yields control to `ga_agent.evolve()`, which internally manages population evaluation and returns the number of consumed steps.

### `runs/`

Directory of generated artifacts. Structure defined in ADR 0006. Contains no code.

```bash
runs/
├── registry.json
├── experiment_main/
│   ├── ga/
│   ├── dqn/
│   ├── ppo/
│   └── random/
└── experiment_robustness/
```

### `analysis/`

Post-experiment statistical analysis scripts.

```bash
analysis/
├── aggregate.py        — builds comparative table from registry
├── plots.py            — learning curves, std bands
├── stats.py            — Shapiro-Wilk, t-test, Mann-Whitney U
└── report.py           — generates final markdown report
```

They do not modify any artifacts in `runs/`. They only read and produce visualizations and tables.

### `configs/`

Configuration files per method and experiment.

```bash
configs/
├── env.yaml            — environment parameters (03_environment_spec)
├── ga.yaml             — GA hyperparameters
├── dqn.yaml            — DQN hyperparameters
├── ppo.yaml            — PPO hyperparameters
└── experiment.yaml     — seeds, budget, frequencies
```

Each run generates a `config.json` snapshot in its directory that freezes the exact configuration used.

### `scripts/`

Entry-point scripts to run experiments.

```bash
scripts/
├── run_experiment.py   — launches full or partial experiment
├── run_single.py       — launches a specific run
└── analyze.py          — executes full analysis pipeline
```

---

## Separation of Responsibilities

| Component   | Knows about         | Does not know about         |
| ----------- | ------------------- | --------------------------- |
| `env/`      | physics, Gymnasium  | agents, logger              |
| `models/`   | PyTorch             | environment, agents, logger |
| `agents/`   | models, env API     | logger, runner              |
| `runner`    | env, agents, logger | internal physics            |
| `logging/`  | filesystem          | env, agents, runner         |
| `analysis/` | filesystem          | env, agents, runner         |

---

## Key Interfaces

### Environment → Agent

$$\left(s_t \in \mathbb{R}^5\right) ;\longrightarrow;
\text{Agent.select\_action} ;\longrightarrow;
\left(a_t \in {0, 1}\right)$$

### Agent → Environment

$$a_t ;\longrightarrow; \text{env.step}(a_t) ;\longrightarrow;
\left(s_{t+1},; r_t,; \texttt{terminated},;
\texttt{truncated},; \texttt{info}\right)$$

### Runner → Logger

Every $10{,}000$ steps:

$$\left(\text{step},; r_t,; \bar{R}*t,;
t*{\text{wall}},; \ldots\right) ;\longrightarrow;
\texttt{logs.csv}$$

At completion:

$$\left(R_{\text{final}},; S_T,; \text{AUC},;
\text{converged},; \ldots\right) ;\longrightarrow;
\texttt{metrics.json}$$

---

## Dependency Diagram

```
configs/
    │
    ▼
scripts/run_experiment.py
    │
    ▼
experiments/runner.py
    ├──► env/flappy_env.py ──► models/mlp.py
    ├──► agents/{method}   ──► models/mlp.py
    └──► logging (ADR 0005)
              │
              ▼
         runs/registry.json
              │
              ▼
    analysis/aggregate.py
              │
              ▼
    analysis/stats.py + plots.py
```

Dependencies flow in a single direction. `analysis/` is never imported by `experiments/`. `env/` never imports from `agents/`.
