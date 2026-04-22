# ADR 0001 — Environment Design

## Status

Accepted

---

## Context

It was necessary to decide how to implement the dynamic environment in which GA, DQN, and PPO would be compared, ensuring fair comparability between methods, reproducibility, integration with standard RL libraries, and low development overhead.

The central problem was choosing between building a custom environment or using an existing one.

Active constraints at the time of the decision:

* Limited development time
* Need for direct integration with Stable-Baselines3
* Need for a reproducible and auditable environment
* Avoid introducing uncontrolled physics bugs

---

## Options Considered

### Option 1 — Implement the environment completely from scratch

Advantages: full control over every detail, total flexibility.

Discarded because it introduces the risk of uncontrolled physics bugs, consumes development time that does not contribute to the comparative objective, and makes auditing the dynamics more difficult.

### Option 2 — Use an Atari / ALE environment with visual input

Advantages: standard and well-studied environments.

Discarded because it requires CNNs, shifts the focus of the project toward perception instead of optimization, increases training variance, and complicates comparison with GA which does not naturally process images.

### Option 3 — Use a different standard environment (CartPole, LunarLander, MountainCar)

Advantages: very well documented, easy integration.

Discarded because they are either too simple or too heavily studied, do not represent a dynamic environment with progressive obstacles, and do not generate the type of emergent behavior relevant for this study.

### Option 4 — Use an existing Flappy Bird environment compatible with Gymnasium

Advantages: standard interface, direct integration with Stable-Baselines3, observations as a feature vector, basic physics already implemented and verifiable.

**Selected.**

---

## Decision

An existing Flappy Bird environment compatible with Gymnasium is used, with feature vector observations of dimension 5, not pixels.

The main reason is to maximize experimental control and comparability between methods without spending effort rebuilding basic physics, and to allow direct integration with the RL libraries used in the project.

A visual version was not used because it introduces unnecessary complexity that shifts the focus of the comparative study toward perception. Flappy Bird from older unmaintained Gymnasium versions was also not used because they may contain undocumented differences and dynamics that are difficult to audit.

---

## Consequences

### Positive

* Direct integration with Stable-Baselines3 without custom code
* Standard Gymnasium interface shared by all methods
* Lower risk of uncontrolled physics errors
* More time available for experimental design
* Easier reproducibility
* All methods interact under exactly the same conditions

### Negative

* Dependence on external design
* Possible implicit decisions not visible in the API
* Less flexibility to deeply redesign physics
* Radically changing the game mechanics would be costly

### Impact on other components

GA can interact with the environment exactly the same way as DQN and PPO, eliminating the need to build a custom interface. This directly improves comparability between methods by reducing biases introduced by custom implementations.

DQN and PPO can use Stable-Baselines3 directly without additional wrappers, reducing custom code and compatibility errors.
