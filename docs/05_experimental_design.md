# 05 — Experimental Design

All conditions are defined before running experiments. They are not modified after seeing results.

---

## Total Budget

A uniform budget is defined for all methods:

$$\text{Budget} = 2{,}000{,}000 \text{ environment steps per run}$$

The unit of comparison is environment steps, not episodes, generations, or network updates. This guarantees direct comparability between GA, DQN, PPO, and Random.

$2\text{M}$ was chosen because pilot tests showed:

* DQN begins stabilizing between $800\text{k}$–$1.2\text{M}$ steps
* PPO between $500\text{k}$–$1\text{M}$ steps
* GA requires more population evaluations to mature

$2\text{M}$ allows observation of full convergence, phases of stagnation, and post-convergence stability without prohibitive computational cost.

---

## Seeds

### Official seeds

Exactly these 10 seeds will be used for all methods:

$$\mathcal{S} = {11,\ 23,\ 37,\ 41,\ 53,\ 67,\ 79,\ 83,\ 97,\ 101}$$

### Assignment per run

For each seed $s \in \mathcal{S}$:

$$\texttt{env\_seed} = s, \qquad \texttt{agent\_seed} = s + 1000$$

This decouples the stochasticity of the environment from that of the algorithm.

### Evaluation seed

All evaluation phases use:

$$\texttt{eval\_seed} = 999$$

Fixed and shared for all methods and all runs. This guarantees that the final comparison occurs in exactly the same environment.

---

## Phases per Run

Each run has two phases:

**Phase 1 — Training**

Duration: $2\text{M}$ environment steps.
The method updates its parameters normally according to its paradigm.

**Phase 2 — Final evaluation**

After the budget ends, evaluation is performed with exploration disabled over $N_{\text{eval}} = 50$ episodes with $\texttt{eval\_seed} = 999$. The metrics from this phase are used for the final comparison between methods (see `06_metrics_definition.md`).

### Intermediate evaluation

During training, every $50{,}000$ steps an intermediate evaluation is performed:

$$\text{Frequency} = \frac{\text{Budget}}{40} = 50{,}000 \text{ steps}$$

* Duration: $10$ episodes
* Deterministic mode, without parameter updates
* Training pauses and then continues
* Used only for monitoring and curve logging
* Not used for final metrics

---

## Protocol by Method

### Random

One run = random interaction until consuming $2\text{M}$ steps. No learning. No parameter updates. Serves as a lower bound of performance.

### GA

One run = full evolution until consuming $2\text{M}$ environment steps, regardless of the number of generations reached.

$$\text{steps}*g = g \times N*{\text{pop}} \times \bar{T}_{\text{ep}}$$

Generations continue until the budget is exhausted.

### DQN

One run = continuous training with replay buffer and target network until $2\text{M}$ steps. Normal updates according to the frequency defined in `04_methodology/dqn.md`.

### PPO

One run = continuous training with rollouts and clipped updates until $2\text{M}$ steps. Update frequency defined in `04_methodology/ppo.md`.

### Artifacts saved per run

For each run of each method:

```bash
runs/
  {method}/
    seed_{s}/
      config.json       — exact snapshot of configuration
      logs.csv          — metrics per step
      model.pt          — final weights
      metrics.json      — summary of final metrics
```

---

## Variability Control

### Within the same method

| Variable        | Status |
| --------------- | ------ |
| Architecture    | Fixed  |
| Hyperparameters | Fixed  |
| Budget          | Fixed  |
| Environment     | Fixed  |
| Seed            | Varies |

### Between different methods

| Variable                 | Status |
| ------------------------ | ------ |
| Environment              | Fixed  |
| Budget (2M steps)        | Fixed  |
| Base seeds $\mathcal{S}$ | Fixed  |
| Evaluation frequency     | Fixed  |
| Metrics                  | Fixed  |
| Hardware                 | Fixed  |
| Algorithm                | Varies |

$$\textbf{The only independent variable is the method.}$$

Hyperparameters are not adjusted by seed or by method
after observing results.

---

## Robustness Tests

After the main experiment, each trained model is evaluated under environment perturbations. Evaluation only, no retraining.

| Perturbation      | Value                             |
| ----------------- | --------------------------------- |
| Increased gravity | $g' = g \times 1.1$               |
| Reduced gap       | $\delta' = \delta \times 0.9$     |
| Increased speed   | $v' = v_{\text{pipe}} \times 1.1$ |

The results of this phase are additional robustness analysis.
They are not part of the main comparison metrics.

---

## Execution Order

Methods run sequentially to avoid resource contention and variation due to CPU load.

Fixed order:

$$\text{GA} \rightarrow \text{DQN} \rightarrow \text{PPO}
\rightarrow \text{Random}$$

Within each method, the 10 seeds run sequentially in ascending order.

---

## Logging and Checkpoints

### Logging during training

Every $10{,}000$ steps the following are recorded:

* Current step
* Recent average reward ($W = 100$ episodes)
* Loss (DQN and PPO)
* Accumulated time in seconds
* Intermediate evaluation result (if it corresponds to the $50\text{k}$ checkpoint)

### Model checkpoints

Saved every:

$$250{,}000 \text{ steps}$$

Each checkpoint includes:

* Model weights
* Optimizer state
* RNG state

### Format

* `logs.csv` — metrics per step, compatible with pandas
* `metrics.json` — final summary per run
* TensorBoard compatible
* `config.json` — exact snapshot of all parameters used
