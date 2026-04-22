# ADR 0004 — DQN Implementation Choice

## Status

Accepted

---

## Context

It was necessary to decide how to implement the project's value-based RL agent, choosing the algorithm variant, implementation library, and base configuration, while maintaining strict comparability with GA and PPO under the same budget of $2\text{M}$ environment steps.

Active constraints at the time of the decision:

* Fixed network architecture (ADR 0002)
* Same environment interface for all methods (ADR 0001)
* Strict comparability with GA and PPO
* Not using variants that introduce uncontrolled structural advantages
* Direct integration with Stable-Baselines3

---

## Options Considered

### Option 1 — From-scratch implementation

Advantages: full control over every detail.

Discarded because it introduces risk of uncontrolled bugs, consumes development time that does not contribute to the comparative objective, and makes reproducibility harder. SB3 is an audited and widely verified implementation.

### Option 2 — Double DQN

Advantages: reduces Q-value overestimation.

Discarded because it introduces an algorithmic improvement that would specifically favor DQN, breaking fair comparison between paradigms. The goal is to compare classical DQN as a representative of the value-based paradigm, not its optimized variant.

### Option 3 — Dueling DQN

Advantages: separates state value and advantage estimation.

Discarded for the same reason as Double DQN: it introduces an uncontrolled architectural advantage that would invalidate the comparison.

### Option 4 — Rainbow

Advantages: combines multiple improvements over base DQN.

Discarded because it combines several extensions simultaneously, making it impossible to isolate what contributes to performance. It directly contradicts the goal of comparing pure paradigms.

### Option 5 — Prioritized Experience Replay (PER)

Advantages: improves sample efficiency by prioritizing transitions.

Discarded because it introduces an advantage in sample efficiency that does not exist in GA or standard PPO, biasing the efficiency comparison which is one of the study’s main metrics.

### Option 6 — SARSA or tabular Q-learning

Advantages: simpler, no neural network.

Discarded because the state space is continuous $s \in \mathbb{R}^5$ and cannot be efficiently discretized. Additionally, they would not use the shared base architecture, breaking comparability.

### Option 7 — Classical DQN via Stable-Baselines3

Advantages: audited implementation, direct integration with Gymnasium, configurable architecture, standard replay buffer, standard target network, standard epsilon-greedy policy.

**Selected.**

---

## Decision

Classical DQN implemented via Stable-Baselines3 is used, with network architecture forced to match the one defined in ADR 0002, epsilon-greedy policy configuration with linear decay, standard-size replay buffer, and target network with periodic updates.

The variant is base DQN without extensions. This positions it as a clean representative of the value-based off-policy paradigm, directly comparable with GA (population-based) and PPO (policy-gradient on-policy).

### Key components

**Replay buffer:**

Stores transitions $(s, a, r, s', \text{done})$ and samples minibatches for updates:

$$\mathcal{D} = {(s_t, a_t, r_t, s_{t+1}, d_t)}$$

Introduces experience reuse that GA and PPO do not have. This is a characteristic of the off-policy paradigm, not an artificial advantage, and it is explicitly stated in the analysis.

**Target network:**

Separate network $\theta^-$ that is periodically updated to stabilize bootstrapping:

$$y_t = r_t + \gamma \max_{a'} Q_{\theta^-}(s_{t+1}, a')$$

**Exploration policy:**

Epsilon-greedy with linear decay from $\varepsilon_{\text{start}}$ to $\varepsilon_{\text{end}}$ throughout training. Exact values are defined in `04_methodology/dqn.md`.

**Implementation:**

Stable-Baselines3 with a custom architecture forced to match ADR 0002 via the `policy_kwargs` parameter.

---

## Consequences

### Positive

* Classical DQN is the standard representative of the value-based off-policy paradigm, widely recognized in the literature
* Audited implementation reduces risk of bugs
* Direct integration with Gymnasium without additional wrappers
* Architecture enforceable via `policy_kwargs` guarantees comparability with GA and PPO
* The replay buffer is a defining characteristic of the paradigm, not an unfair advantage

### Negative

* Not using Double DQN means Q-value overestimation may occur, which can affect stability
* The replay buffer introduces a lag between collected data and data used for updates, which does not exist in PPO
* Without PER, all transitions have equal weight, which can make learning slower in early stages

---

### Impact on other components

The replay buffer causes DQN to consume $2\text{M}$ steps differently from PPO and GA: it reuses past transitions, so the number of gradient updates is not proportional to the number of steps in the same way. This is explicitly stated in `04_methodology/comparison_framework.md` and controlled by using environment steps as the common unit, not gradient updates.

For logging, DQN generates additional metrics that GA does not have: network loss, mean Q-value, current epsilon. These are recorded in `logs.csv` but are not part of the main comparison metrics defined in `06_metrics_definition.md`.

The DQN learning curve may show oscillations due to bootstrapping that GA does not exhibit. This is expected behavior of the paradigm and is analyzed as part of the study, not as an artifact to be corrected.
