# 09 — Results Template

This document defines which plots, tables, and statistical analyses will be produced before running the experiments. The metrics are defined in `06_metrics_definition.md`. The comparison framework is defined in `04_methodology/comparison_framework.md`.

No results are included here. This file is filled with actual values after running the experiments. What is defined now is the exact structure those results will have.

---

## Table 1 — Final Performance by Method

Metric: $R_{\text{final}}$ over $N_{\text{eval}} = 50$ episodes with `eval_seed = 999`, averaged over $N = 10$ seeds.

| Method | $\mu$ | $\sigma$ | $\text{CI}_{95\%}$ | $\max$ | $\min$ |
| ------ | ----- | -------- | ----------------- | ------ | ------ |
| Random | —     | —        | —                 | —      | —      |
| GA     | —     | —        | —                 | —      | —      |
| DQN    | —     | —        | —                 | —      | —      |
| PPO    | —     | —        | —                 | —      | —      |

Expected interpretation: PPO and DQN should outperform GA in $R_{\text{final}}$. Random should remain close to zero. See hypotheses in `00_research_question.md`.

---

## Table 2 — Stability Across Seeds

Metric: coefficient of variation $\text{CV} = \sigma / \mu$ over $R_{\text{final}}^{(i)}$ for $i = 1, \ldots, 10$.

| Method | $\mu$ | $\sigma$ | $\text{CV}$ | Stability ranking |
| ------ | ----- | -------- | ----------- | ----------------- |
| Random | —     | —        | —           | —                 |
| GA     | —     | —        | —           | —                 |
| DQN    | —     | —        | —           | —                 |
| PPO    | —     | —        | —           | —                 |

Expected interpretation: PPO should show lower CV than DQN. GA may show high variance in early seeds.

---

## Table 3 — Sample Efficiency

Metric: $S_T$ = steps to reach $\bar{R}_t \geq T = 500$. For methods that do not reach $T$, report $S_T = \infty$.

| Method | $\mu(S_T)$ | $\sigma(S_T)$ | Success ($N/10$) |
| ------ | ---------- | ------------- | ---------------- |
| Random | —          | —             | —                |
| GA     | —          | —             | —                |
| DQN    | —          | —             | —                |
| PPO    | —          | —             | —                |

Expected interpretation: PPO and DQN should reach $T$ in fewer steps than GA. Random should not reach it.

---

## Table 4 — Area Under the Curve

Metric: $\text{AUC} = \sum_{t=1}^{T_{\text{budget}}} \bar{R}_t$ over the full budget of $2\text{M}$ steps.

| Method | $\mu(\text{AUC})$ | $\sigma(\text{AUC})$ | $\text{CI}_{95\%}$ |
| ------ | ----------------- | -------------------- | ----------------- |
| Random | —                 | —                    | —                 |
| GA     | —                 | —                    | —                 |
| DQN    | —                 | —                    | —                 |
| PPO    | —                 | —                    | —                 |

AUC captures both convergence speed and final performance quality. A method that converges quickly and maintains good performance will have higher AUC than one that converges late even if it reaches the same level.

---

## Table 5 — Training Time

Metric: wall-clock time in seconds averaged over $N = 10$ seeds.

| Method | $\mu(t)$ sec | $\sigma(t)$ | $t_{\min}$ | $t_{\max}$ |
| ------ | ------------ | ----------- | ---------- | ---------- |
| Random | —            | —           | —          | —          |
| GA     | —            | —           | —          | —          |
| DQN    | —            | —           | —          | —          |
| PPO    | —            | —           | —          | —          |

Time comparison is indicative. The main efficiency metric is environment steps, not compute time. See `06_metrics_definition.md`.

---

## Table 6 — Statistical Comparison Between Methods

Results of statistical tests on $R_{\text{final}}^{(i)}$ for each pair of methods. Protocol defined in `06_metrics_definition.md`.

| Pair          | Test used | Statistic | p-value | Significant ($\alpha=0.05$) |
| ------------- | --------- | --------- | ------- | --------------------------- |
| PPO vs DQN    | —         | —         | —       | —                           |
| PPO vs GA     | —         | —         | —       | —                           |
| PPO vs Random | —         | —         | —       | —                           |
| DQN vs GA     | —         | —         | —       | —                           |
| DQN vs Random | —         | —         | —       | —                           |
| GA vs Random  | —         | —         | —       | —                           |

The test used (t-test or Mann-Whitney U) is determined after applying Shapiro-Wilk to each distribution of $R_{\text{final}}^{(i)}$.

---

## Table 7 — Robustness to Perturbations

Metric: $R_{\text{final}}$ under perturbed evaluation, compared with $R_{\text{final}}$ under normal conditions.

Relative degradation:

$$\Delta R = \frac{R_{\text{normal}} - R_{\text{perturb}}}{R_{\text{normal}}}$$

| Method | $R_{\text{normal}}$ | $R_{g \times 1.1}$ | $\Delta R_g$ | $R_{\delta \times 0.9}$ | $\Delta R_\delta$ | $R_{v \times 1.1}$ | $\Delta R_v$ |
| ------ | ------------------- | ------------------ | ------------ | ----------------------- | ----------------- | ------------------ | ------------ |
| Random | —                   | —                  | —            | —                       | —                 | —                  | —            |
| GA     | —                   | —                  | —            | —                       | —                 | —                  | —            |
| DQN    | —                   | —                  | —            | —                       | —                 | —                  | —            |
| PPO    | —                   | —                  | —            | —                       | —                 | —                  | —            |

Lower $\Delta R$ indicates higher robustness.

---

## Plot 1 — Learning Curves

**What it shows:** $\mu_t \pm \sigma_t$ of average episode reward over $N = 10$ seeds, for all four methods on the same axis.

**X-axis:** Environment steps $\in [0, 2{,}000{,}000]$

**Y-axis:** $\bar{R}_t$ (moving average $W = 100$)

**Elements:**

* One line per method with different color
* Shaded band $\mu_t \pm \sigma_t$
* Dashed horizontal line at $T = 500$
* Legend with method names

**What to observe:**

* Takeoff speed of each method
* Smoothness vs oscillations of the curve
* Point where each method crosses $T = 500$
* Width of the variance band

---

## Plot 2 — Learning Curves with Median and IQR

Same as Plot 1 but using median and percentiles $[25, 75]$ instead of mean and std. Produced as a robust alternative to outliers across seeds.

---

## Plot 3 — Boxplot of $R_{\text{final}}$

**What it shows:** Distribution of $R_{\text{final}}^{(i)}$ over $N = 10$ seeds per method.

**X-axis:** Method (Random, GA, DQN, PPO)

**Y-axis:** $R_{\text{final}}$

**Elements:**

* Box with Q1, median, Q3
* Whiskers up to $1.5 \times \text{IQR}$
* Individual points for outliers
* Dashed horizontal line at $T = 500$

**What to observe:**

* Distribution spread per method
* Presence of outliers
* Overlap between methods

---

## Plot 4 — Boxplot of $S_T$ (Steps to Threshold)

**What it shows:** Distribution of steps to reach $T = 500$ over $N = 10$ seeds per method.

**X-axis:** Method

**Y-axis:** $S_T$ in environment steps

**Note:** Seeds that did not reach $T$ are marked as $\infty$ and excluded from the boxplot, with success counts reported separately.

---

## Plot 5 — Robustness Comparison

**What it shows:** $\Delta R$ per method and per perturbation type.

**Type:** Grouped bar chart.

**X-axis:** Perturbation type ($g \times 1.1$,
$\delta \times 0.9$, $v \times 1.1$)

**Y-axis:** $\Delta R$ (relative degradation)

**Grouping:** One bar per method per perturbation.

**What to observe:**

* Which method degrades less
* Whether certain perturbations affect some paradigms more

---

## Plot 6 — GA Fitness Curves by Generation

**What it shows:** Evolution of best fitness and mean population fitness per generation, averaged over $N = 10$ seeds.

**X-axis:** Generation $g$

**Y-axis:** Fitness

**Elements:**

* Best fitness line with band $\mu \pm \sigma$
* Mean fitness line with band $\mu \pm \sigma$

**Purpose:** Internal GA diagnostics. Allows observing whether the population converges or maintains diversity. Not a cross-method comparison plot.

---

## Expected Narrative Analysis

The final report in `analysis/report.py` will follow this structure:

**Section 1 — Baseline validation**

Does Random behave as expected? Does it confirm the environment is not trivially solvable?

**Section 2 — Performance comparison**

Which method achieves higher $R_{\text{final}}$? Is the difference statistically significant according to Table 6?

**Section 3 — Efficiency comparison**

Which method reaches $T = 500$ with fewer steps? Are there methods that never reach it?

**Section 4 — Stability comparison**

Which method shows lower CV across seeds? Are there failed seeds in any method?

**Section 5 — Robustness analysis**

Which method degrades less under perturbations? Is there correlation between seed stability and robustness?

**Section 6 — Hypothesis discussion**

Compare each hypothesis from `00_research_question.md` with the obtained results. For each hypothesis: confirmed, partially confirmed, or rejected.

**Section 7 — Limitations**

Refer to `11_threats_to_validity.md`.
