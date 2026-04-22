# 10 — Future Work

This document enumerates extensions that were explicitly scoped out of the main study but are natural next steps on top of its infrastructure. Each item includes what it would add, what part of the current pipeline it would reuse, and what decisions it would force to reopen.

The criterion for inclusion is that the extension is compatible with the existing comparison framework without invalidating the methodology defined in `00_research_question.md` and `05_experimental_design.md`.

---

## Advanced Algorithmic Variants

### NEAT (NeuroEvolution of Augmenting Topologies)

Evolves both weights and network topology instead of fixing the architecture a priori as ADR 0002 does.

- Reuses: environment, evaluation budget, fitness logic of the current GA.
- Reopens: ADR 0002 (architecture comparability), because NEAT by construction does not respect a fixed parameter count.
- Expected value: decoupling the GA hypothesis from the MLP architecture, isolating the contribution of evolving topology.

### Rainbow DQN

DQN variant that integrates Double DQN, Dueling Networks, Prioritized Experience Replay, Multi-step Learning, Distributional RL, and Noisy Nets.

- Reuses: environment, complete logging infrastructure, evaluation protocol.
- Reopens: ADR 0004 (DQN implementation decision), because the current baseline is classic DQN via SB3.
- Expected value: measuring how much of the DQN vs. PPO gap is explained by PPO being "more modern" rather than by a paradigm-level difference.

### SAC (Soft Actor-Critic)

Off-policy algorithm with explicit entropy regularization. Requires adapting the environment to continuous action space or using a discrete variant (SAC-Discrete).

- Reuses: environment state, reward function, logging infrastructure.
- Reopens: ADR 0001 (discrete action design) if the continuous version is used.
- Expected value: closing the gap between value-based (DQN) and on-policy (PPO) with a modern off-policy method.

---

## More Complex Input Representations

### Visual Input with CNN

Replaces the current $5$-dimensional engineered state with raw pixel frames.

- Reuses: environment dynamics, budget and seed protocol, aggregation scripts.
- Reopens: ADR 0002 (MLP architecture), comparability with GA (which does not scale well to high-dimensional input).
- Expected value: quantifying the penalty paid by each paradigm for not having a hand-engineered representation.

### Frame Stacking and Recurrent Policies

Input that includes the last $k$ frames, or policies with recurrent memory (LSTM, GRU).

- Reuses: environment, evaluation protocol.
- Reopens: `02_problem_definition.md` (changes observability, moves from fully-observable MDP to POMDP in some formulations).
- Expected value: studying how methods behave under partial observability.

---

## Multi-Agent Extensions

### Cooperative Multi-Agent

Two or more agents in the same environment with a shared or coupled reward signal.

- Reuses: base environment implementation, metric infrastructure.
- Reopens: almost all of `02_problem_definition.md`, `03_environment_spec.md`, `05_experimental_design.md`, and the entire `04_methodology/` because most algorithms require multi-agent variants (MADDPG, QMIX, CTDE).
- Expected value: high, but requires a project of comparable scope to this one.

### Competitive (Self-Play)

Agents trained against previous versions of themselves.

- Reuses: agent infrastructure, logging, directory structure.
- Reopens: the very concept of "seed", since self-play introduces a non-stationarity that is not fully captured by a single seed.
- Expected value: connecting with literature on adversarial training and convergence of equilibrium policies.

---

## Curriculum and Transfer Learning

### Curriculum Learning

The environment gradually increases difficulty (obstacle spacing, gravity, speed) as the agent improves.

- Reuses: parameterized environment, logging, config schema.
- Reopens: `05_experimental_design.md` to define the curriculum schedule and how it interacts with the fixed budget.
- Expected value: testing whether curriculum significantly changes the ranking between paradigms.

### Transfer Learning Across Environments

Training in environment $A$ and evaluating or fine-tuning in environment $B$.

- Reuses: agents, logging.
- Reopens: the whole `03_environment_spec.md` because it requires defining at least a second environment.
- Expected value: beyond the scope of this repository as a single project, but natural if a second compatible environment is built.

---

## Deeper Statistical Analysis

### Sensitivity Analysis over Hyperparameters

Currently each method uses a single hyperparameter configuration per ADR. A sensitivity analysis would sweep key hyperparameters ($\eta$, $\gamma$, population size, $\epsilon$ schedule) with a reduced budget and report performance surfaces.

- Reuses: runner, logging, aggregate script.
- Reopens: nothing conceptual, only requires more compute budget.
- Expected value: supports or attenuates the claim that one method is better than another "with reasonable hyperparameters".

### Bootstrap Confidence Intervals

Replaces the simple standard deviation between seeds with bootstrap confidence intervals, following current best practice in deep RL (Henderson et al. 2018, Colas et al. 2018).

- Reuses: `aggregate.py`, `metrics.json`.
- Reopens: `06_metrics_definition.md` to formalize the new intervals.
- Expected value: higher statistical rigor in reported conclusions.

---

## Pipeline Improvements

### Continuous Integration

Automation of quality and reproducibility tests on every push: `uv sync`, Docker builds, short training runs (few steps) as smoke tests.

- Reuses: `Makefile`, Docker images.
- Reopens: nothing.
- Expected value: guarantees that the state published in `main` is always reproducible.

### Automatic Report Generation

Pipeline that, given a completed experiment, produces the full final report (plots, tables, statistical analysis) in a reproducible way.

- Reuses: `aggregate.py`, plotting scripts.
- Reopens: `09_results_template.md` to bind the template to the generated output format.
- Expected value: eliminates the manual step of writing the report, closing the reproducibility loop completely.

---

## Out of Scope Forever

For clarity, extensions explicitly out of scope of this project family:

- Formal theoretical convergence analysis of evolutionary or RL methods.
- Energy efficiency or hardware cost comparisons as a primary metric.
- Wrappers around third-party implementations without methodological contribution.
- Visual demos of the trained agent as the project deliverable: this project is a methodological comparison, not a playable demo.
