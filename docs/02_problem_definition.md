# 02 — Problem Definition

## Formal Definition of the MDP

The environment is modeled as a Markov Decision Process:

$$\mathcal{M} = (S, A, P, R, \gamma)$$

---

## State Space

The state is defined as a continuous vector of dimension 5:

$$s = \left[ y_t,\ v_t,\ d_t,\ g_t,\ \Delta_t \right] \in \mathbb{R}^5$$

where:

* $y_t$ — current height of the agent
* $v_t$ — vertical velocity
* $d_t$ — horizontal distance to the next obstacle
* $g_t$ — height of the center of the obstacle gap
* $\Delta_t = g_t - y_t$ — relative vertical difference

All variables are continuous and normalized to a fixed range for numerical stability.

$n=5$ was chosen because it keeps the state fully observable under the current design, avoids redundancy, and provides sufficient information for an optimal policy without requiring additional memory.

---

## Action Space

The action space is discrete and binary:

$$A = {0, 1}$$

* $a = 0$ — do nothing
* $a = 1$ — apply positive vertical impulse (jump)

$$|A| = 2$$

---

## Reward Function

$$
R(s, a, s') =
\begin{cases}
+1 & \text{if the agent survives the step} \
-100 & \text{if a collision occurs}
\end{cases}
$$

The cumulative return per episode is:

$$G_t = \sum_{k=0}^{T} \gamma^k , r_{t+k}$$

The reward structure incentivizes prolonged survival. The collision penalty is asymmetric to strongly discourage crashing behavior.

---

## Discount Factor

$$\gamma = 0.99$$

It favors long-term planning, maintains training stability, and slightly penalizes extremely distant rewards.

---

## Transitions

Local dynamics are deterministic:

$$v_{t+1} = v_t + g + a \cdot J$$
$$y_{t+1} = y_t + v_{t+1}$$

where $g$ is the constant gravitational acceleration and $J$ is the
fixed jump impulse.

The movement of existing obstacles is also deterministic.

Stochasticity arises exclusively from the procedural generation of new obstacles when the previous one leaves the field of view: gap height and vertical spacing are sampled randomly.

$$P(s'|s,a) \text{ is partially stochastic}$$

---

## Observability

The agent directly observes $s_t \in \mathbb{R}^5$.

The state vector contains all the information necessary to make optimal decisions under the current design. No historical memory or hidden state is required.

$$\textbf{Fully observable MDP}$$

---

## Episode

An episode begins with a fixed or slightly randomized initial position and the first generated obstacle.

It ends when:

$$\text{collision} ;\lor; y_t \notin [y_{\min}, y_{\max}]$$

Optionally, a maximum step limit $T_{\max}$ can be imposed to avoid episodes of indefinite duration in agents that converge to very strong policies.

---

## Stochasticity

| Source                      | Type          | Affects          |
| --------------------------- | ------------- | ---------------- |
| Obstacle gap height         | Stochastic    | Transition       |
| Gap vertical separation     | Stochastic    | Transition       |
| Initial environment seed    | Controlled    | Reproducibility  |
| Physical dynamics (gravity) | Deterministic | Local transition |
| Obstacle movement           | Deterministic | Local transition |

The environment is stochastic in transition but deterministic in local dynamics. This implies that two episodes with the same seed produce the same sequence of obstacles, which is necessary for fair comparison between methods.

---

## Problem Interpretation by Method

### Random

Selects the action uniformly at each step:

$$a_t \sim \text{Uniform}({0, 1})$$

It does not optimize any parameter. It does not update any internal state. It serves as a lower bound of performance.

### GA

It does not interact with the MDP in the classical RL sense. It directly optimizes the weights of the neural network:

$$\theta^* = \arg\max_{\theta} ; \mathbb{E}[G(\theta)]$$

The fitness of an individual is the cumulative return of the full episode:

$$\text{fitness}(\theta) = G(\theta)$$

There is no parameter update within the episode. The learning signal is the full episode evaluated as a scalar.

### DQN

Approximates the action-value function:

$$Q_\theta(s, a) \approx \mathbb{E}[G_t \mid s_t = s,; a_t = a]$$

Optimizes the Bellman loss:

$$
L(\theta) = \mathbb{E}!\left[
\left(
Q_\theta(s,a) -
\left(r + \gamma \max_{a'} Q_{\theta^-}(s', a')\right)
\right)^2
\right]
$$

where $\theta^-$ are the parameters of the target network, updated periodically.

### PPO

Directly optimizes the policy $\pi_\theta(a \mid s)$ by maximizing the clipped objective:

$$
\mathcal{L}^{\text{CLIP}}(\theta) = \mathbb{E}!\left[
\min!\left(
r_t(\theta),\hat{A}_t,;
\text{clip}(r_t(\theta),, 1-\epsilon,, 1+\epsilon),\hat{A}_t
\right)
\right]
$$

where the probability ratio is:

$$r_t(\theta) = \frac{\pi_\theta(a_t \mid s_t)}{\pi_{\theta_{\text{old}}}(a_t \mid s_t)}$$

and $\hat{A}_t$ is the advantage estimate at timestep $t$.
