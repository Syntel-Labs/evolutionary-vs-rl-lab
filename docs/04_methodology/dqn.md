# 04 — Methodology: DQN

This document describes the DQN methodology as a representative of the value-based off-policy paradigm. The decision to use classical DQN via Stable-Baselines3 is justified in ADR 0004. The network architecture is defined in ADR 0002. The experimental protocol is defined in `05_experimental_design.md`.

---

## Paradigm

DQN approximates the state-action value function using a neural network:

$$Q_\theta(s, a) \approx \mathbb{E}[G_t \mid s_t = s,; a_t = a]$$

Unlike GA, DQN learns step by step by reusing past experience stored in a replay buffer. Unlike PPO, it is off-policy: training data does not need to come from the current policy.

---

## Main Components

### Neural network

MLP architecture defined in ADR 0002:

$$\text{Input}(5) \to \text{Linear}(5, 64) \to \text{ReLU}
\to \text{Linear}(64, 64) \to \text{ReLU}
\to \text{Linear}(64, 2)$$

The output layer produces one Q-value per action:

$$Q_\theta(s, \cdot) \in \mathbb{R}^{|A|} = \mathbb{R}^2$$

Enforced via `policy_kwargs` in SB3 to match exactly the shared architecture.

### Replay Buffer

Stores transitions $(s_t, a_t, r_t, s_{t+1}, d_t)$:

$$\mathcal{D} = {(s_t, a_t, r_t, s_{t+1}, d_t)}_{t=1}^{|\mathcal{D}|}$$

| Parameter          | Value                                   |
| ------------------ | --------------------------------------- |
| Maximum capacity   | $100{,}000$ transitions                 |
| Replacement policy | FIFO                                    |
| Warm-up steps      | $1{,}000$ steps before the first update |

The replay buffer introduces experience reuse that GA and PPO do not have. This is a defining characteristic of the off-policy paradigm, not an artificial advantage. It is explicitly stated in `04_methodology/comparison_framework.md`.

### Target Network

Separate network $\theta^-$ that stabilizes bootstrapping:

$$y_t = r_t + \gamma \max_{a'} Q_{\theta^-}(s_{t+1}, a')$$

It is updated every $\tau_{\text{update}}$ steps by copying the weights from the main network:

$$\theta^- \leftarrow \theta$$

| Parameter              | Value           |
| ---------------------- | --------------- |
| $\tau_{\text{update}}$ | $1{,}000$ steps |

No soft updates (hard update).

---

## Loss Function

The Bellman error is minimized over minibatches sampled from $\mathcal{D}$:

$$L(\theta) = \mathbb{E}*{(s,a,r,s',d) \sim \mathcal{D}}!\left[
\left(
Q*\theta(s, a) - y
\right)^2
\right]$$

where the target is:

$$y = r + (1 - d) \cdot \gamma \cdot \max_{a'} Q_{\theta^-}(s', a')$$

and $d \in {0, 1}$ indicates whether the episode has terminated.

The discount factor $\gamma = 0.99$ is consistent with the one defined in `02_problem_definition.md`.

---

## Exploration Policy

Epsilon-greedy with linear decay is used:

$$a_t =
\begin{cases}
\arg\max_{a} Q_\theta(s_t, a) & \text{with prob. } 1 - \varepsilon_t \
a \sim \text{Uniform}({0, 1}) & \text{with prob. } \varepsilon_t
\end{cases}$$

The decay is linear from $\varepsilon_{\text{start}}$ to
$\varepsilon_{\text{end}}$ over $\varepsilon_{\text{decay}}$ steps:

$$\varepsilon_t = \varepsilon_{\text{start}} -
\frac{(\varepsilon_{\text{start}} - \varepsilon_{\text{end}}) \cdot t}
{\varepsilon_{\text{decay}}}$$

| Parameter                    | Value             |
| ---------------------------- | ----------------- |
| $\varepsilon_{\text{start}}$ | $1.0$             |
| $\varepsilon_{\text{end}}$   | $0.05$            |
| $\varepsilon_{\text{decay}}$ | $500{,}000$ steps |

During evaluation: $\varepsilon = 0$ (pure greedy policy), consistent with `06_metrics_definition.md`.

---

## Update

### Frequency

One gradient update every $\Delta_{\text{update}}$ steps:

$$\Delta_{\text{update}} = 4 \text{ steps}$$

### Minibatch

At each update, a minibatch is sampled from $\mathcal{D}$:

$$\mathcal{B} \sim \text{Uniform}(\mathcal{D}), \qquad
|\mathcal{B}| = 32$$

### Optimizer

$$\text{Adam}(\theta, \alpha = 10^{-4})$$

| Parameter     | Value     |
| ------------- | --------- |
| Learning rate | $10^{-4}$ |
| Batch size    | $32$      |
| Gradient clip | $10$      |

---

## Training Loop

```bash id="z3b4fn"
initialize network θ, target θ⁻ = θ, buffer D
obs = env.reset(env_seed)

while steps < 2_000_000:
    if steps < warm_up:
        action = random
    else:
        action = ε-greedy(Q_θ, obs, ε_t)

    obs_next, reward, terminated, truncated, _ = env.step(action)
    D.add(obs, action, reward, obs_next, terminated or truncated)
    obs = obs_next
    steps += 1

    if terminated or truncated:
        obs = env.reset(env_seed)

    if steps >= warm_up and steps % Δ_update == 0:
        B = D.sample(batch_size)
        y = r + (1 - d) · γ · max_a Q_θ⁻(s', a)
        L = MSE(Q_θ(s, a), y)
        θ ← Adam(θ, ∇L)

    if steps % τ_update == 0:
        θ⁻ ← θ

    if steps % 50_000 == 0:
        evaluate with ε=0, eval_seed=999, 10 episodes
```

---

## Hyperparameters

| Parameter                    | Value             |
| ---------------------------- | ----------------- |
| $\gamma$                     | $0.99$            |
| Learning rate                | $10^{-4}$         |
| Batch size                   | $32$              |
| Replay buffer capacity       | $100{,}000$       |
| Warm-up steps                | $1{,}000$         |
| $\Delta_{\text{update}}$     | $4$ steps         |
| $\tau_{\text{update}}$       | $1{,}000$ steps   |
| $\varepsilon_{\text{start}}$ | $1.0$             |
| $\varepsilon_{\text{end}}$   | $0.05$            |
| $\varepsilon_{\text{decay}}$ | $500{,}000$ steps |
| Gradient clip                | $10$              |
| Implementation               | SB3 `DQN`         |

All hyperparameters are fixed across all seeds. They are not tuned between runs nor after observing results.

---

## Intermediate Evaluation

Every $50{,}000$ steps, the greedy policy ($\varepsilon = 0$) is evaluated over 10 episodes with `eval_seed = 999`. The result is recorded in `logs.csv` under the `eval_reward` field.

This evaluation temporarily pauses training and does not count against the step budget.

---

## DQN-specific Logging

In addition to the common fields defined in ADR 0005, DQN logs
every $10{,}000$ steps:

| Field     | Description                                    |
| --------- | ---------------------------------------------- |
| `loss`    | Value of $L(\theta)$ in the last update        |
| `epsilon` | Current value of $\varepsilon_t$               |
| `q_mean`  | Mean of $\max_a Q_\theta(s, a)$ over the batch |

These fields are not part of the main comparison metrics defined in `06_metrics_definition.md`. They are informative for training diagnostics.

---

## Expected Behavior

Consistent with the hypotheses in `00_research_question.md`:

* Faster convergence than GA in early stages due to experience reuse via replay buffer
* Oscillations in the learning curve due to bootstrapping with a target network
* Sensitivity to the exploration phase: low performance while $\varepsilon$ is high, rapid improvement as it decays
* Possible degradation if the replay buffer becomes saturated with early suboptimal policy transitions
* Occasional instability due to Q-value overestimation, a consequence of not using Double DQN (see ADR 0004)
