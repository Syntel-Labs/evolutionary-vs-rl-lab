# 04 — Methodology: Genetic Algorithm

This document describes the GA methodology as an evolutionary optimization method. The chromosome representation is defined in ADR 0003. The network architecture is defined in ADR 0002. The experimental protocol is defined in `05_experimental_design.md`.

---

## Paradigm

The GA does not interact with the environment in the classical RL sense. It does not update parameters within an episode nor use gradients.

It directly optimizes the policy through population evolution:

$$\theta^* = \arg\max_{\theta} ; \mathbb{E}[G(\theta)]$$

The learning signal is the accumulated return of the full episode evaluated as a scalar. There is no bootstrap, no replay buffer, and no value function.

---

## Representation

Each individual is a flat continuous vector in `float32`:

$$\theta \in \mathbb{R}^d, \qquad
d = (5 \times 64 + 64) + (64 \times 64 + 64) + (64 \times 2 + 2)$$

The mapping between chromosome and network is deterministic and reversible. See ADR 0003 for justification of this representation.

---

## Population Structure

### Size

$$N_{\text{pop}} = 100 \text{ individuals}$$

### Initialization

Each individual is initialized independently:

$$\theta_i \sim \mathcal{N}(0, \sigma_{\text{init}}^2),
\qquad \sigma_{\text{init}} = 0.1$$

No pretrained initialization. No weight transfer.

---

## Fitness Function

For each individual $\theta_i$ in each generation:

1. The network is built with weights $\theta_i$
2. A full episode is executed with the current run’s `env_seed`
3. The total return is accumulated without discount

$$\text{fitness}(\theta_i) = \sum_{t=0}^{T} r_t$$

where $r_t \in {+1, -100}$ according to the reward function defined in `02_problem_definition.md`.

No additional transformation is applied to the fitness. It is not normalized across individuals within the same generation.

---

## Selection

Binary tournament selection is used:

1. Sample $k = 3$ individuals at random from the population
2. The individual with the highest fitness wins the tournament
3. Repeat until the required parents are obtained

$$\text{winner} = \arg\max_{\theta \in \text{tournament}} \text{fitness}(\theta)$$

Tournament selection with $k=3$ provides moderate selective pressure, avoiding premature convergence that occurs with large $k$.

---

## Elitism

The top $E$ individuals from each generation are preserved without modification:

$$E = \lfloor 0.05 \times N_{\text{pop}} \rfloor = 5$$

Elites are copied directly into the next generation. They do not participate in crossover or mutation in that generation. They do participate as candidates in parent selection.

---

## Crossover

### Type

Uniform crossover over the weight vector:

For each gene $i$ of the offspring chromosome:

$$\theta_{\text{child}}^{(i)} =
\begin{cases}
\theta_{\text{parent1}}^{(i)} & \text{with prob. } 0.5 \
\theta_{\text{parent2}}^{(i)} & \text{with prob. } 0.5
\end{cases}$$

### Crossover probability

$$p_c = 0.8$$

With probability $1 - p_c$, the offspring is a direct copy of parent 1.

### Number of offspring per generation

$$N_{\text{offspring}} = N_{\text{pop}} - E = 95$$

Each pair of parents produces one offspring.

---

## Mutation

### Type

Gaussian mutation applied gene-wise with probability $p_m$:

$$\theta_i \leftarrow \theta_i + \varepsilon, \qquad
\varepsilon \sim \mathcal{N}(0, \sigma_{\text{mut}}^2)$$

### Parameters

$$p_m = \frac{1}{d} \approx 0.002, \qquad \sigma_{\text{mut}} = 0.05$$

$p_m = 1/d$ is the standard rate for real-valued representation: on average, one gene is mutated per individual per generation.

### Application

Mutation is applied to offspring after crossover. Elites are not mutated.

---

## Generational Cycle

Each generation follows this exact order:

```bash id="5r1k3p"
1. Evaluate fitness of the entire population (N_pop episodes)
2. Identify E elites
3. Select parents via tournament (N_offspring × 2 tournaments)
4. Perform crossover → N_offspring offspring
5. Mutate offspring with probability p_m per gene
6. New population = elites ∪ offspring
7. Log best fitness and accumulated steps
8. Check budget → if steps ≥ Budget: STOP
```

### Step count per generation

$$\text{steps}*g = N*{\text{pop}} \times \bar{T}_{\text{ep}}^{(g)}$$

where $\bar{T}_{\text{ep}}^{(g)}$ is the average episode length in generation $g$. The total budget is controlled by accumulating these steps until reaching $2\text{M}$.

---

## Stopping Criterion

The GA stops when:

$$\sum_{g=0}^{G} \text{steps}_g \geq 2{,}000{,}000$$

There is no early convergence criterion. The GA consumes the full budget regardless of the achieved fitness. This guarantees comparability with DQN and PPO, which also consume the full $2\text{M}$ steps.

---

## Hyperparameters

| Parameter              | Value | Description                      |
| ---------------------- | ----- | -------------------------------- |
| $N_{\text{pop}}$       | 100   | Population size                  |
| $E$                    | 5     | Elite individuals per generation |
| $k$                    | 3     | Tournament size                  |
| $p_c$                  | 0.8   | Crossover probability            |
| $p_m$                  | $1/d$ | Mutation probability per gene    |
| $\sigma_{\text{init}}$ | 0.1   | Initialization std               |
| $\sigma_{\text{mut}}$  | 0.05  | Gaussian mutation std            |

All hyperparameters are fixed across all seeds. They are not tuned between runs nor after observing results.

---

## Intermediate Evaluation

Every $50{,}000$ accumulated steps, the best individual in the current population is evaluated deterministically over 10 episodes with `eval_seed = 999`. This result is recorded in `logs.csv` under the `eval_reward` field.

This evaluation does not modify the population nor count against the step budget.

---

## GA-specific Logging

In addition to the common fields defined in ADR 0005, GA logs per generation:

| Field          | Description                             |
| -------------- | --------------------------------------- |
| `generation`   | Current generation number               |
| `best_fitness` | Fitness of the best individual          |
| `mean_fitness` | Mean fitness of the population          |
| `std_fitness`  | Standard deviation of fitness           |
| `step`         | Accumulated steps up to this generation |

The `step` field uses the conversion defined in `06_metrics_definition.md` so that the GA curve is comparable with DQN and PPO on the same horizontal axis.

---

## Expected Behavior

Consistent with the hypotheses in `00_research_question.md`:

* Gradual improvement generation by generation without oscillations
* High variance across seeds in early generations
* Broad exploration of the parameter space in initial phases
* Slower refinement compared to gradient-based methods
* Robustness to environment stochasticity due to no reliance on gradients or bootstrap
* Slower convergence in terms of environment steps compared to PPO and DQN
