# 04 — Methodology: Comparison Framework

This document defines how the four methods are compared against each other. The metrics are defined in `06_metrics_definition.md`. The execution protocol is defined in `05_experimental_design.md`. This document answers only: what is held constant, what changes, and how observed differences are interpreted.

---

## Purpose

This is not a results document. It is the framework that defines the conditions under which results will be interpreted.

Without this framework, comparing GA with DQN and PPO would be unfair because each paradigm has a different natural unit of progress:

| Method | Natural unit of progress |
| ------ | ------------------------ |
| GA     | Generation               |
| DQN    | Gradient update          |
| PPO    | Rollout epoch            |
| Random | Step                     |

The comparison is made on a common unit:

$$\textbf{Environment steps} \quad (\text{same for all})$$

---

## What Is Held Constant

The following conditions are identical for all methods across all runs:

| Variable             | Fixed value                           |
| -------------------- | ------------------------------------- |
| Environment          | Flappy Bird, Gymnasium API            |
| State space          | $s \in \mathbb{R}^5$                  |
| Action space         | $A = {0, 1}$                          |
| Reward function      | $R \in {+1, -100}$                    |
| Budget               | $2{,}000{,}000$ environment steps     |
| Base seeds           | $\mathcal{S} = {11, 23, \ldots, 101}$ |
| Eval seed            | $\texttt{eval_seed} = 999$            |
| Base architecture    | MLP $5 \to 64 \to 64 \to 2$           |
| Number of runs       | $N = 10$ seeds per method             |
| Evaluation frequency | Every $50{,}000$ steps                |
| Comparison metrics   | Defined in `06_metrics_definition.md` |
| Hardware             | Same machine, sequential execution    |

$$\textbf{The only independent variable is the method.}$$

---

## What Changes per Method

The following characteristics are inherent to each paradigm and are not controlled across methods because they define the paradigm:

| Characteristic        | GA                             | DQN                        | PPO         | Random  |
| --------------------- | ------------------------------ | -------------------------- | ----------- | ------- |
| Uses gradients        | No                             | Yes                        | Yes         | No      |
| Reuses experience     | No                             | Yes (buffer)               | No          | No      |
| Learns within episode | No                             | Yes                        | Yes         | No      |
| Update unit           | Generation                     | Step                       | Rollout     | —       |
| Exploration           | Population                     | $\varepsilon$-greedy       | Entropy     | Uniform |
| Updates per step      | 0 (batch at end of generation) | $1/\Delta_{\text{update}}$ | Per rollout | 0       |

These differences are the subject of the study. They are not threats to validity; they are the object of analysis.

---

## Common Unit of Comparison

### The problem

GA consumes steps in blocks of $N_{\text{pop}} \times \bar{T}_{\text{ep}}$ per generation. DQN consumes them one by one with updates every 4 steps. PPO consumes them in fixed-size rollouts.

### The solution

All methods report their progress on the same axis:

$$\text{horizontal axis} = \text{accumulated environment steps}$$

For GA, the conversion is:

$$\text{steps}*g = \sum*{i=0}^{g} N_{\text{pop}} \times \bar{T}_{\text{ep}}^{(i)}$$

For DQN and PPO, the axis is directly the environment step counter, incremented by 1 for each `env.step()`.

This ensures that the learning curves of all four methods are comparable on the same plot.

---

## Declared Structural Differences

The following differences between methods are known, explicitly declared, and are not considered biases but characteristics of the paradigm:

### DQN replay buffer

DQN reuses past transitions. GA and PPO do not. This makes DQN more sample-efficient in terms of gradient updates per step, but introduces a lag between collected experience and experience used for learning.

The comparison in environment steps partially controls for this: all methods consume the same number of interactions with the environment, regardless of how many times each method reuses those interactions internally.

### Number of gradient updates per step

| Method | Gradient updates per environment step |
| ------ | ------------------------------------- |
| GA     | 0 (does not use gradients)            |
| DQN    | $1 / \Delta_{\text{update}} = 0.25$   |
| PPO    | Variable per rollout                  |
| Random | 0                                     |

This asymmetry is inherent to the paradigms and is declared in the analysis. It is not artificially corrected.

### DQN warm-up

DQN requires $1{,}000$ warm-up steps before the first update. During this period it behaves like Random. This is reflected in the learning curve and is explicitly declared in the analysis.

---

## Comparison Protocol

### Main comparison

The four methods are compared using the metrics defined in `06_metrics_definition.md`:

$${R_{\text{final}},; S_T,; \text{AUC},; \text{CV}}$$

Each metric is computed over the $N = 10$ seeds for each method. Statistical comparison uses Shapiro-Wilk + t-test or Mann-Whitney U depending on normality, with $\alpha = 0.05$.

### Curve comparison

Learning curves are plotted on the same environment steps axis with band $\mu_t \pm \sigma_t$ over $N = 10$ seeds.

The analysis order is:

1. Does Random consistently achieve $R > 0$? If not, the environment is not functioning correctly.
2. Does any method surpass threshold $T = 500$? Which one first?
3. Which converges earlier in terms of steps?
4. Which is more stable across seeds (lower CV)?
5. Which achieves higher $R_{\text{final}}$?

### Robustness comparison

After the main experiment, trained models are evaluated under environment perturbations defined in `05_experimental_design.md`.

The analysis order is:

1. Which method degrades less under $g' = g \times 1.1$?
2. Which method degrades less under $\delta' = \delta \times 0.9$?
3. Is there correlation between seed stability and robustness to perturbations?

---

## Interpretation of Results

### Valid claims

Given the scope of the study defined in `00_research_question.md`, the following claims are valid:

* "Method X converged faster than Y in this environment under this step budget"
* "Method X showed lower variance across seeds than Y"
* "Method X was more robust to perturbation Z"

### Invalid claims

The following claims are not supported by this study:

* "Method X is better than Y in general"
* "Method X will converge faster in any environment"
* "These results generalize to continuous or visual environments"

---

## Threats to Comparison

The following threats are known and mitigated as indicated:

| Threat                     | Mitigation                          |
| -------------------------- | ----------------------------------- |
| Hyperparameter bias        | Equal tuning effort for all methods |
| Unequal architecture       | Identical backbone (ADR 0002)       |
| Different seeds per method | Same $\mathcal{S}$ for all          |
| Different progress units   | Environment steps as common unit    |
| Stochastic variability     | $N = 10$ seeds, $CI_{95\%}$ reported |
| Asymmetric DQN warm-up     | Explicitly declared in analysis     |
| DQN replay buffer          | Declared as paradigm characteristic |

Non-mitigable threats are declared as limitations in `11_threats_to_validity.md`.
