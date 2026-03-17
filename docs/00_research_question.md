# 00 — Research Question

## Central Question

How do different optimization paradigms behave when they must learn a survival policy in a stochastic dynamic environment with discrete action, under a controlled budget of interactions?

The four compared methods are:

| Method | Paradigm        | Type             | Role       |
| ------ | --------------- | ---------------- | ---------- |
| Random | None            | Minimal baseline | Control    |
| GA     | Evolutionary    | Population-based | Evolution  |
| DQN    | Value-based RL  | Off-policy       | Classic RL |
| PPO    | Policy-based RL | On-policy        | Modern RL  |

Comparison conditions:

* Same dynamic Flappy Bird–type environment
* Same base architecture (comparable MLP in number of parameters)
* Same state and action space
* Same budget of interactions with the environment
* Multiple seeds (N=10)

The question is NOT "which algorithm is better in the abstract".
It is an empirical comparison under controlled conditions and a fixed budget.

---

## "Best" is operationally defined as

* Higher final average reward (performance)
* Lower variance across seeds (stability)
* Fewer interactions to reach a defined threshold (efficiency)

The maximum isolated score is NOT the main metric.

---

## Hypotheses

### Random (baseline)

* Performance close to zero in all episodes.
* No improvement over time.
* Serves to verify that the environment is not trivially solvable by chance.

### GA

* Consistent but slow improvement in terms of total interactions.
* High variance between runs if the population is small.
* Robustness to environment stochasticity by not depending on gradients.
* Lower sensitivity to reward shaping than RL methods.
* Transfer from prior work (TSP): GA explores the parameter space well but takes time to refine. In noisy environments it can be more robust than gradient-based methods, but less sample-efficient.

### DQN

* Faster convergence than GA in early stages due to reuse
  of experience (replay buffer).
* Greater instability: oscillations due to bootstrapping.
* Sensitive to hyperparameters and exploration policy (epsilon).
* It may degrade if the replay buffer is not sufficiently diverse.

### PPO

* More stable than DQN due to policy clipping.
* Better expected final average performance.
* Smoother convergence.
* Higher computational cost per update compared to DQN.

### Conditions where one would outperform another

| Condition                           | Expected advantage |
| ----------------------------------- | ------------------ |
| Low interaction budget              | PPO > DQN > GA     |
| High budget + high noise            | GA may get closer  |
| Poor reward shaping                 | GA less affected   |
| Stable and well-defined environment | PPO or DQN         |

---

## Independent Variables

Main:

* Method (Random, GA, DQN, PPO)

Explicitly controlled sub-variables:

* Network architecture (fixed, comparable in parameters across all)
* Population size (GA)
* Learning rate (DQN, PPO)
* Replay buffer size (DQN)
* Batch size
* Random seed (same N seeds for all methods)

---

## Dependent Variables

| Metric                       | Description                                  |
| ---------------------------- | -------------------------------------------- |
| Average reward per episode   | Moving average over a window of 100 episodes |
| Best reward achieved         | Historical maximum per run                   |
| Interactions until threshold | Steps to exceed a defined score threshold    |
| Variance across seeds        | Std over 10 independent runs                 |
| Real training time           | Wall-clock seconds per method                |

### Formal definition of convergence

A method is considered converged when:

    Moving_average(reward, window=100) remains within $\pm$5% for at least 500 consecutive episodes.

Or alternatively when it exceeds a fixed performance threshold T (defined in 06_metrics_definition.md before running experiments).

The exact definition is fixed before experimenting and is not modified after seeing the results.

---

## Scope and Delimitations of the Study

* State of the art in RL or evolutionary algorithms is not evaluated.
* Globally optimal hyperparameters are not sought for any method.
* Energy efficiency or hardware cost is not compared.
* Generalization to other environments or games is not evaluated.
* Formal theoretical convergence is not studied.
* Advanced variants are not compared (NEAT, SAC, Rainbow, etc.).

The study is specific to this environment with this state representation.

---

## Threats to Validity

### 1. Hyperparameter bias

A method could appear better only because it was better tuned.

Mitigation: same search effort for all, hyperparameters explicitly reported in 04_methodology/.

### 2. Definition of the comparison budget

The unit of comparison is total interactions with the environment, not episodes or computation time. This is the fairest choice because all methods depend on the environment as the source of signal.

### 3. Architecture comparability

If GA uses a larger network than PPO, the comparison is not fair. The number of parameters is kept comparable across all methods that use neural networks.

### 4. Stochastic variability

Results with few seeds can be misleading. N=10 fixed seeds are used and are identical for all methods.

### 5. Generalization

The results are not extrapolated to:

* Environments with continuous state spaces or high-dimensional visual input
* Multi-agent games
* Environments with very sparse rewards

This must be explicitly stated in the conclusions.

---

## Final statement

This project is:

    An empirical comparison between evolutionary optimization, value-based reinforcement learning, and policy-gradient methods under controlled interaction budgets in a dynamic discrete-action survival environment.

It is not a general benchmark. It is a controlled experiment with fixed conditions, predefined metrics, and statistical analysis across multiple runs.
