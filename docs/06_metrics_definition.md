# 06 — Metrics Definition

All metrics are defined before running experiments. They are not modified after seeing results.

---

## Convergence

Let $R_t$ be the average reward of episode $t$.

The moving average over window $W$ is:

$$\bar{R}*t = \frac{1}{W} \sum*{i=t-W+1}^{t} R_i \qquad W = 100$$

A method is considered **converged** when:

$$\left| \frac{\bar{R}*t - \bar{R}*{t-K}}{\bar{R}_{t-K}} \right| \leq \epsilon
\qquad \epsilon = 0.05,\quad K = 200$$

for $K = 200$ consecutive episodes.

A fixed performance threshold is also defined:

$$T = 500 \text{ steps of average survival per episode}$$

If the agent consistently exceeds $T$, the environment is considered solved.

If a method does not converge within the total interaction budget:

* It is reported as **not converged**
* Its final performance is used for comparison
* The total interactions consumed are recorded

> Note: $T$ is a performance metric (average reward), not the environment limit. The environment limit is $T_{\max} = 10000$ defined in 03_environment_spec.md.

---

## Performance

The final reward is computed in a **separate evaluation phase**, with exploration disabled, over $N_{\text{eval}} = 50$ episodes:

$$R_{\text{final}} = \frac{1}{N_{\text{eval}}} \sum_{i=1}^{N_{\text{eval}}} R_i$$

Evaluation conditions by method:

| Method | Evaluation mode                |
| ------ | ------------------------------ |
| GA     | Network without noise          |
| DQN    | Greedy policy ($\epsilon = 0$) |
| PPO    | Deterministic policy           |
| Random | No changes                     |

The environment uses a dedicated fixed evaluation seed: $\texttt{eval\_seed} = 999$, different from the training seeds. This seed is the same for all methods during evaluation.

---

## Efficiency (Sample Efficiency)

The primary unit of comparison is:

$$\text{Total environment steps}$$

Not episodes. Not generations. This makes GA, DQN, and PPO directly comparable regardless of paradigm.

A method **achieved the objective** if:

$$\bar{R}_t \geq T \qquad T = 500$$

If it never reaches $T$, the following are reported:

* Total interactions consumed
* Best reward achieved
* Binary success indicator: $\mathbb{1}[\bar{R}_t \geq T] = 0$

---

## Stability

For each method, $N = 10$ independent seeds are executed, obtaining $R_{\text{final}}^{(i)}$ for $i = 1, \ldots, 10$.

The following are reported:

$$\mu = \frac{1}{N}\sum_{i=1}^{N} R_{\text{final}}^{(i)}, \qquad
\sigma = \text{std}\left(R_{\text{final}}^{(i)}\right)$$

$$\text{IC}_{95\%} = \mu \pm 1.96 \cdot \frac{\sigma}{\sqrt{N}}$$

Relative stability is quantified as:

$$\text{CV} = \frac{\sigma}{\mu}$$

A method is more stable if $\text{CV}$ is smaller.

Stability is measured across three dimensions:

1. $R_{\text{final}}$ — reward during evaluation
2. $S_T$ — steps until reaching threshold $T$
3. $\text{AUC}$ — area under the learning curve

$$\text{AUC} = \sum_{t=1}^{T_{\text{budget}}} \bar{R}_t$$

---

## Training Time

Real wall-clock time is measured from start to completion, without rendering, on the same hardware for all methods.

For GA, the time includes the entire evolutionary process:

$$t_{\text{GA}} = t_{\text{eval}} + t_{\text{crossover}} +
t_{\text{mutation}} + t_{\text{2opt}} + t_{\text{species}}$$

Time comparison is indicative. The main efficiency metric is environment steps, not computation time.

---

## Statistical Comparison

To compare two methods $A$ and $B$ over their $N = 10$ runs:

1. Normality test: Shapiro-Wilk on $R_{\text{final}}^{(i)}$
2. If normal $\rightarrow$ paired t-test
3. If not normal $\rightarrow$ Mann–Whitney U

Significance level:

$$\alpha = 0.05$$

With $N = 10$ seeds, the results are sufficient for reasonable comparative evidence in an academic context, but not for top-tier paper-level claims. This limitation is explicitly stated in the conclusions.

Always report:

$$\mu \pm \sigma \quad \text{and} \quad \text{IC}_{95\%}$$

Never the mean alone.

---

## Aggregation Across Runs

### RL Methods (DQN, PPO, Random)

For each step $t$, the aggregated curve is computed as:

$$\mu_t = \frac{1}{N} \sum_{i=1}^{N} R_t^{(i)}, \qquad
\sigma_t = \text{std}!\left(R_t^{(i)}\right)$$

The horizontal axis is directly $t$ in environment steps.

### GA

GA does not learn step by step but generation by generation. The curve is expressed in generations $g$ and converted to accumulated steps to plot on the same axis as RL methods:

$$\text{steps}*g = g \times N*{\text{pop}} \times \bar{T}_{\text{ep}}$$

where:

* $N_{\text{pop}}$ — population size
* $\bar{T}_{\text{ep}}$ — average episode length in that generation

The value plotted at each point is the fitness of the best individual of generation $g$, mapped to the accumulated steps axis.

### Common aggregation

For all methods, once on the environment steps axis:

$$\mu_t = \frac{1}{N} \sum_{i=1}^{N} R_t^{(i)}, \qquad
\sigma_t = \text{std}!\left(R_t^{(i)}\right)$$

$\mu_t$ is plotted with shaded band $\mu_t \pm \sigma_t$.

As a robust alternative to outliers, the following is also reported:

$$\text{median}*t \pm \text{IQR}*{[25, 75]}$$

Outliers are not manually removed. A run is excluded only if there was a technical error, crash, or incorrect configuration, never due to poor performance. Any exclusion is explicitly documented.
