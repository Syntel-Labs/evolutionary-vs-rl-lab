# 11 — Threats to Validity

This document catalogs all known threats to the validity of the study. High-level threats are declared in `00_research_question.md`. This document expands them with formal categorization and mitigation status for each.

Not all threats are mitigable. Non-mitigable ones are explicitly stated so the reader can calibrate the scope of the conclusions.

---

## Categories

Four standard categories of validity in experimental research are used:

* **Internal validity**: Are the observed differences truly due to the method and not another variable?
* **External validity**: Do the results generalize beyond this experiment?
* **Construct validity**: Do the metrics actually measure what they claim to measure?
* **Statistical validity**: Are the conclusions statistically supported?

---

## Internal Validity

### V-I-1 — Hyperparameter bias

**Description:** A method could appear better simply because its hyperparameters were better tuned than others.

**Potential impact:** High. It can invert the ranking of methods.

**Mitigation:** The same tuning effort is applied to all methods. Hyperparameters are explicitly reported in `04_methodology/` and in `configs/`. They are not adjusted after observing results.

**Status:** Partially mitigated. No exhaustive hyperparameter search was performed. The values used are standard in the literature for each paradigm.

---

### V-I-2 — Architecture comparability

**Description:** If one method uses a larger or more expressive network, it could achieve better results due to capacity rather than paradigm.

**Potential impact:** High. Invalidates direct comparison.

**Mitigation:** Identical backbone (MLP $5 \to 64 \to 64$) for GA, DQN, and PPO. Enforced via `policy_kwargs` in SB3. Defined in ADR 0002.

**Status:** Mitigated. The shared backbone guarantees capacity comparability. The only difference in output heads (PPO has actor and critic) is minimal and declared.

---

### V-I-3 — Asymmetric DQN warm-up

**Description:** DQN requires $1{,}000$ warm-up steps before the first update. During this period it behaves like Random, consuming budget without learning.

**Potential impact:** Low. $1{,}000$ steps is $0.05%$ of the budget.

**Mitigation:** Explicitly declared in `04_methodology/dqn.md` and in `04_methodology/comparison_framework.md`.

**Status:** Not structurally mitigable. It is a defining characteristic of the off-policy paradigm. Declared as a limitation.

---

### V-I-4 — DQN experience reuse

**Description:** DQN reuses past transitions via a replay buffer. GA and PPO do not. This gives DQN more gradient updates per environment step.

**Potential impact:** Medium. It may favor DQN in sample efficiency.

**Mitigation:** Comparison is done in environment steps, not gradient updates. This is the fairest approach since all methods depend on the environment as the signal source. Declared in `04_methodology/comparison_framework.md`.

**Status:** Partially mitigated. The common step unit controls interaction budget, but not parameter update count. This asymmetry is inherent to the off-policy paradigm and is explicitly declared.

---

### V-I-5 — Stochastic variability with N=10 seeds

**Description:** With only 10 seeds, results may be sensitive to the specific seed selection.

**Potential impact:** Medium. May lead to conclusions that would not hold with more seeds.

**Mitigation:** Fixed and identical seeds for all methods ($\mathcal{S} = {11, 23, 37, 41, 53, 67, 79, 83, 97, 101}$). $\text{CI}_{95\%}$ is reported for all metrics. Statistical tests use $\alpha = 0.05$.

**Status:** Partially mitigated. $N = 10$ is sufficient for reasonable comparative evidence in an academic context, but insufficient for top-tier paper-level claims. Explicitly stated in `06_metrics_definition.md`.

---

### V-I-6 — Contamination between evaluation and training phases

**Description:** If intermediate evaluation influenced training, results would be biased.

**Potential impact:** Low.

**Mitigation:** Intermediate evaluation pauses training, uses independent `eval_seed = 999`, does not update parameters, and does not count against the budget. Defined in `05_experimental_design.md`.

**Status:** Mitigated.

---

### V-I-7 — Sequential execution order

**Description:** Methods run sequentially. If hardware degrades over time (temperature, memory), later methods could be affected.

**Potential impact:** Low. The order is fixed and known.

**Mitigation:** Order documented in `05_experimental_design.md`. The order is GA → DQN → PPO → Random. If systematic degradation exists, it would affect PPO and Random more than GA and DQN.

**Status:** Not fully mitigable. The exact order is declared so the reader can account for this factor.

---

## External Validity

### V-E-1 — Generalization to other environments

**Description:** Results are specific to the Flappy Bird environment with state representation $s \in \mathbb{R}^5$.

**Potential impact:** High for generalization. Does not affect internal validity.

**Mitigation:** Scope is explicitly stated in `00_research_question.md` and `01_project_overview.md`.

**Status:** Not mitigable. This is a declared limitation. Results do not generalize to continuous, visual, multi-agent, or sparse-reward environments.

---

### V-E-2 — Generalization to other implementations of the same paradigm

**Description:** Classical DQN and standard PPO are used. Other variants (Double DQN, PPO with curiosity, etc.) could produce different results for the same paradigm.

**Potential impact:** Medium. Conclusions apply to these specific implementations, not the paradigms in abstract.

**Mitigation:** The chosen variants are the most standard and representative of each paradigm. Justified in ADR 0004.

**Status:** Partially mitigated. Claims are made about specific implementations, not paradigms in general.

---

### V-E-3 — Generalization to other step budgets

**Description:** With a different budget (smaller or larger), the ranking of methods could change.

**Potential impact:** Medium. With a small budget, GA may lag more. With a very large budget, it might catch up to RL methods.

**Mitigation:** The $2\text{M}$ budget was chosen to observe full convergence of all methods based on pilot experiments. Declared in `05_experimental_design.md`.

**Status:** Not fully mitigable. Conclusions are valid under this specific budget.

---

## Construct Validity

### V-C-1 — Definition of convergence

**Description:** Convergence is defined as moving average stability within $\pm 5%$ over $K = 200$ episodes. This is arbitrary; other definitions could classify methods differently.

**Potential impact:** Medium. Affects metric $S_T$.

**Mitigation:** Definition is fixed before experiments and not modified after observing results. Formally declared in `06_metrics_definition.md`.

**Status:** Partially mitigated. Arbitrariness is inherent to any operational definition of convergence.

---

### V-C-2 — Threshold T as proxy for solving

**Description:** $T = 500$ average survival steps is defined as the solution threshold. This value is arbitrary.

**Potential impact:** Medium. A higher threshold could result in no method reaching it; a lower one could make all methods reach it trivially.

**Mitigation:** Threshold is chosen before experiments based on environment analysis. Full curves are also reported, allowing independent interpretation.

**Status:** Partially mitigated.

---

### V-C-3 — AUC as combined efficiency metric

**Description:** AUC combines convergence speed and final performance into a single number. A method that converges quickly to a low level may have higher AUC than one that converges slowly to a high level.

**Potential impact:** Low. AUC is complementary, not primary.

**Mitigation:** AUC is reported alongside $R_{\text{final}}$ and $S_T$, allowing disaggregated analysis.

**Status:** Mitigated by report design.

---

## Statistical Validity

### V-S-1 — Statistical power with N=10

**Description:** With $N = 10$ runs per method, statistical power to detect small effects is limited.

**Potential impact:** Medium. Real but small differences may not be statistically significant.

**Mitigation:** Effect size is reported alongside p-values. $\text{CI}_{95\%}$ communicates uncertainty.

**Status:** Not fully mitigable with $N = 10$. Explicitly stated that this is insufficient for top-tier claims.

---

### V-S-2 — Multiple comparisons

**Description:** Six pairwise comparisons are performed (Table 6 in `09_results_template.md`). Without correction, the probability of at least one false positive increases.

**Potential impact:** Medium. With $\alpha = 0.05$ and 6 comparisons, family-wise error rate is $\approx 26%$ without correction.

**Mitigation:** Bonferroni correction is applied if multiple significant comparisons exist:

$$\alpha_{\text{corrected}} = \frac{0.05}{6} \approx 0.0083$$

**Status:** Mitigated by protocol.

---

### V-S-3 — Distribution normality

**Description:** Parametric tests (t-test) assume normality. With $N = 10$, normality cannot be verified with high confidence.

**Potential impact:** Low. Shapiro-Wilk is used to choose between parametric and non-parametric tests.

**Mitigation:** Adaptive protocol defined in `06_metrics_definition.md`: Shapiro-Wilk first, t-test if normal, Mann-Whitney U otherwise.

**Status:** Mitigated by protocol.

---

## Summary

| ID    | Category    | Impact | Status              |
| ----- | ----------- | ------ | ------------------- |
| V-I-1 | Internal    | High   | Partially mitigated |
| V-I-2 | Internal    | High   | Mitigated           |
| V-I-3 | Internal    | Low    | Declared            |
| V-I-4 | Internal    | Medium | Partially mitigated |
| V-I-5 | Internal    | Medium | Partially mitigated |
| V-I-6 | Internal    | Low    | Mitigated           |
| V-I-7 | Internal    | Low    | Declared            |
| V-E-1 | External    | High   | Declared            |
| V-E-2 | External    | Medium | Partially mitigated |
| V-E-3 | External    | Medium | Declared            |
| V-C-1 | Construct   | Medium | Partially mitigated |
| V-C-2 | Construct   | Medium | Partially mitigated |
| V-C-3 | Construct   | Low    | Mitigated           |
| V-S-1 | Statistical | Medium | Declared            |
| V-S-2 | Statistical | Medium | Mitigated           |
| V-S-3 | Statistical | Low    | Mitigated           |
