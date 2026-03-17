# ADR 0002 — Neural Network Architecture

## Status

Accepted

---

## Context

It was necessary to define a neural network architecture that would be sufficiently expressive to solve the environment, comparable across GA, DQN, and PPO, trainable under the budget of $2\text{M}$ steps, and that would keep the GA search space manageable.

The central problem was balancing capacity, comparability, and computational cost under a low-dimensional input $s \in \mathbb{R}^5$.

Active constraints at the time of the decision:

* Strict comparability between methods
* GA scales worse with more parameters
* PPO and DQN require training stability
* Fixed budget of $2\text{M}$ environment steps

---

## Options Considered

### Option 1 — Small MLP (1 layer, 32 neurons)

Advantages: minimal search space for GA, very fast updates.

Discarded because it may be insufficiently expressive to learn a robust survival policy.

### Option 2 — Deep MLP (3–4 layers)

Advantages: greater representational capacity.

Discarded because it inflates the GA search space to levels where convergence within the budget is unlikely, and increases the update cost in PPO.

### Option 3 — RNN

Advantages: allows explicit temporal memory.

Discarded because the MDP is fully observable with $s \in \mathbb{R}^5$, so historical memory is not required. It also introduces unnecessary complexity and is incompatible with the GA chromosome representation.

### Option 4 — CNN

Advantages: standard for visual environments.

Discarded because the environment uses a feature vector, not pixels. It would introduce perception complexity that diverts the focus of the comparative study.

### Option 5 — Different architecture per method

Advantages: each method could use its optimal architecture.

Discarded because it would introduce an uncontrolled variable (different capacity), invalidating the comparison between methods.

### Option 6 — Stable-Baselines3 default architectures

Advantages: immediate integration without configuration.

Discarded because DQN and PPO have different default configurations in SB3, which would break the strict comparability required.

### Option 7 — 2-layer MLP, 64 neurons per layer

Advantages: sufficient expressiveness, manageable search space for GA, fast updates, compatible with SB3.

**Selected.**

---

## Decision

A 2-hidden-layer MLP with 64 neurons per layer and ReLU activation is used as the common base architecture for GA, DQN, and PPO:

$$\text{Input}(5) ;\to; \text{Linear}(5, 64) ;\to; \text{ReLU}
;\to; \text{Linear}(64, 64) ;\to; \text{ReLU} ;\to;
\text{Output}$$

The output layer varies by method:

| Method | Output head                                                     |
| ------ | --------------------------------------------------------------- |
| GA     | $\text{Linear}(64, |A|)$                                        |
| DQN    | $\text{Linear}(64, |A|)$                                        |
| PPO    | Actor: $\text{Linear}(64, |A|)$, Critic: $\text{Linear}(64, 1)$ |

The backbone (hidden layers) is identical across the three methods.

The activation function is ReLU due to stability, efficiency, compatibility with SB3, and lack of saturation compared to tanh, which especially benefits GA.

Weight initialization is Xavier/Glorot uniform for DQN and PPO, and small-magnitude Gaussian initialization for GA. Both are within the same order of magnitude range to maintain comparability.

---

## Consequences

### Positive

* Strict comparability: the shared backbone has exactly
  the same number of parameters across methods
* Manageable GA search space, enabling convergence
  within the $2\text{M}$ step budget
* Fast updates in PPO and DQN
* Sufficient capacity for an $\mathbb{R}^5$ input environment
  without spatial extraction or long-term memory

### Negative

* Sacrifices maximum absolute performance that could be obtained with
  larger networks in DQN and PPO
* Scaling to visual environments would require complete redesign
* Introducing temporal memory would require an architecture change

### Impact on other components

The number of network parameters directly defines the GA chromosome dimension. With 2 layers of 64 neurons:

$$|\theta| = (5 \times 64 + 64) + (64 \times 64 + 64) + (64 \times |A| + |A|)$$

Maintaining this size is critical for GA to explore the parameter space within the given budget.

For DQN, the architecture does not affect the replay buffer size but does affect the update cost, which remains moderate.

For PPO, it allows multiple epochs per batch without saturating the computation time per rollout generation.
