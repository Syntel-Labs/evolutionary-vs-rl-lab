# 03 — Environment Spec

## Configurable Parameters

All parameters are defined via `config.yaml`.

### Environment

| Parameter       | Default | Description                            |
| --------------- | ------- | -------------------------------------- |
| `gravity`       | 0.5     | Vertical acceleration per step         |
| `jump_impulse`  | -8.0    | Vertical velocity applied when jumping |
| `pipe_speed`    | 3.0     | Horizontal speed of obstacles          |
| `pipe_gap`      | 120 px  | Size of the gap between obstacles      |
| `pipe_spacing`  | 250 px  | Horizontal distance between obstacles  |
| `max_steps`     | 10000   | Maximum number of steps per episode    |
| `y_{\min}`      | 0       | Lower bound of the play space          |
| `y_{\max}`      | 500     | Upper bound of the play space          |
| `screen_width`  | 400 px  | Width of the environment               |
| `screen_height` | 500 px  | Height of the environment              |
| `seed`          | None    | Environment seed                       |

### Parameters that affect difficulty

Increase difficulty:
$\uparrow g$, $\uparrow v_{\text{pipe}}$, $\downarrow \delta_{\text{gap}}$,
$\downarrow d_{\text{spacing}}$, $\downarrow J$

Decrease difficulty:
$\uparrow \delta_{\text{gap}}$, $\downarrow v_{\text{pipe}}$, $\downarrow g$

### Parameters that affect reproducibility

* `seed`
* `pipe_spacing`
* Obstacle generation distribution
* Initial position (if randomization is enabled)

---

## Exact Dynamics

### Gravity

At each step, regardless of the action taken:

$$v_t = v_{t-1} + g$$
$$y_t = y_{t-1} + v_t$$

where $g = 0.5$ is the constant gravitational acceleration.

### Jump impulse

If $a_t = 1$, the vertical velocity is instantaneously replaced:

$$v_t = J \quad \text{where } J = -8.0$$

It does not accumulate over the current velocity. The impulse overwrites $v_t$ before gravity is applied in the next step.

### Obstacle movement

Constant horizontal displacement to the left at each step:

$$x_t^{\text{pipe}} = x_{t-1}^{\text{pipe}} - v_{\text{pipe}}$$

where $v_{\text{pipe}} = 3.0$ px/step.

### Obstacle generation

When an obstacle completely leaves the screen, a new one is generated at $x = w_{\text{screen}}$. The height of the gap center is sampled:

$$g_{\text{center}} \sim \text{Uniform}(100,; y_{\max} - 100)$$

### Gap size

The gap is constant for all obstacles:

$$g_{\text{top}} = g_{\text{center}} - \frac{\delta_{\text{gap}}}{2}$$
$$g_{\text{bottom}} = g_{\text{center}} + \frac{\delta_{\text{gap}}}{2}$$

where $\delta_{\text{gap}} = 120$ px.

---

## Dimensions and Limits

$$x \in [0,; w_{\text{screen}}] = [0, 400]$$
$$y \in [y_{\min},; y_{\max}] = [0, 500]$$

The episode ends if:

$$y_t \leq y_{\min} \quad \lor \quad y_t \geq y_{\max}$$

There is no bounce at the boundaries.

---

## Input Normalization

All variables are statically normalized to $[-1, 1]$ using fixed theoretical maximum limits. The normalization does not depend on dynamic episode statistics.

| Variable            | Transformation                                                    |
| ------------------- | ----------------------------------------------------------------- |
| $y_t$               | $\hat{y} = \dfrac{y_t}{y_{\max}} \cdot 2 - 1$                     |
| $v_t$               | $\hat{v} = \text{clip}!\left(\dfrac{v_t}{v_{\max}}, -1, 1\right)$ |
| $d_t$               | $\hat{d} = \dfrac{d_t}{w_{\text{screen}}} \cdot 2 - 1$            |
| $g_{\text{center}}$ | $\hat{g} = \dfrac{g_{\text{center}}}{y_{\max}} \cdot 2 - 1$       |
| $\Delta_t$          | $\hat{\Delta} = \dfrac{g_{\text{center}} - y_t}{y_{\max}}$        |

where $v_{\max} = 15$ is the theoretical maximum vertical velocity.

---

## Reset Conditions

At the start of each episode:

$$y_0 = \frac{h_{\text{screen}}}{2}, \quad v_0 = 0$$

Obstacle list cleared. Step counter reset to zero.

Randomized initial position (optional):

$$y_0 \sim \text{Uniform}(y_{\min} + 50,; y_{\max} - 50)$$

Controlled by `seed`.

---

## Seeds and Reproducibility

The environment initializes with:

$$\texttt{np.random.seed}(\texttt{seed})$$

With a fixed seed, the following are identical across runs:

* Sequence of generated obstacles
* Values of $g_{\text{center}}$ per obstacle
* Initial position $y_0$ if randomization is active

It is recommended to keep seeds separate:

$$\texttt{env\_seed} \perp \texttt{agent\_seed}$$

This allows the environment to remain deterministic while the agent's exploration remains independent.

---

## Update Frequency

| Mode          | Speed               |
| ------------- | ------------------- |
| Training      | Maximum (no render) |
| Visualization | 30 FPS              |

There is no frameskip by default:

$$\text{1 action} ;\longrightarrow; \text{1 physical step}$$

$\texttt{action\_repeat} = k$ can optionally be enabled.

---

## Interface with Agents

The environment implements the standard Gymnasium API.

**Input per step:**

$$a_t \in {0, 1}$$

**Output per step:**

$$\left(, s_{t+1},; r_t,; \texttt{terminated},; \texttt{truncated},;
\texttt{info} ,\right)$$

where:

$$s_{t+1} \in \mathbb{R}^5, \quad r_t \in \mathbb{R},
\quad \texttt{terminated} = \mathbb{1}[\text{collision}],
\quad \texttt{truncated} = \mathbb{1}[t \geq T_{\max}]$$

Compatible with Stable-Baselines3, Gymnasium, PPO and DQN out of the box.
