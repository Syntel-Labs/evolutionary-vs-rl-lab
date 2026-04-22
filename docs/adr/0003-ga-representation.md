# ADR 0003 — GA Representation

## Status

Accepted

---

## Context

It was necessary to define how to represent a neural policy as an evolutionary individual within the GA, deciding the chromosome form, its mapping to the network, the operators it enables, and its compatibility with the architecture defined in ADR 0002.

The representation directly determines what type of crossover is possible, what type of mutation is valid, how smooth the search space is, and how comparable the GA is with DQN and PPO.

Active constraints at the time of the decision:

* The network architecture is already fixed (ADR 0002)
* The chromosome must be compatible with simple and stable operators
* The search space must not explode unnecessarily
* Population evaluation must be efficient within $2\text{M}$ steps

---

## Options Considered

### Option 1 — Binary representation

Advantages: well studied in classical GA literature.

Discarded because weights are continuous and discretizing into binary introduces artificial quantization, makes the search space rough, and makes fine convergence more difficult.

### Option 2 — Integer representation

Advantages: more resolution than binary.

Discarded because it also discretizes weights, does not reflect the continuous nature of the problem, and introduces abrupt steps in the search.

### Option 3 — Indirect representation (rules, trees, genetic programming)

Advantages: allows evolving symbolic behaviors.

Discarded because it introduces structure as an additional uncontrolled variable, breaks comparability with RL, and complicates the central comparative analysis of the project.

### Option 4 — NEAT (topology evolution)

Advantages: can discover more efficient architectures.

Discarded because it evolves topology and changes the number of parameters dynamically, introducing an uncontrolled structural advantage and preventing direct comparability with PPO and DQN which have fixed architectures.

### Option 5 — Flat continuous vector of weights (float32)

Advantages: direct phenotype representation, smooth continuous space, simple operators, full compatibility with fixed architecture, direct comparability with RL.

**Selected.**

---

## Decision

Each individual is represented as a flat continuous vector of all network weights and biases:

$$\theta \in \mathbb{R}^d$$

where $d$ is the total number of parameters of the network defined in ADR 0002:

$$d = (5 \times 64 + 64) + (64 \times 64 + 64) + (64 \times |A| + |A|)$$

The genotype has no internal structure. It is a 1D continuous vector in `float32`. The structure is reconstructed only when mapping it to the network through a deterministic and reversible process:

1. The flat vector $\theta$ is taken
2. It is segmented according to the size of each layer
3. It is reshaped into weight matrices and bias vectors
4. They are loaded into the neural network

The population is initialized with independently sampled weights:

$$\theta_i \sim \mathcal{N}(0, \sigma^2)$$

with small variance, without pretrained initialization.

### Operators enabled by this representation

Crossover operates over the continuous vector:

* Uniform crossover
* Arithmetic crossover (blend)
* Binary mask crossover
* One-point crossover

Mutation is Gaussian applied to a random subset of genes:

$$\theta_i \leftarrow \theta_i + \varepsilon, \qquad
\varepsilon \sim \mathcal{N}(0, \sigma_{\text{mut}}^2)$$

Elitism is trivial: copy the full vector without repair.

### Fitness computation

For each individual $\theta$:

1. The network is built with weights $\theta$
2. A full episode is executed in the environment
3. Total reward is accumulated

$$\text{fitness}(\theta) = G(\theta) = \sum_{t=0}^{T} r_t$$

Direct evaluation without additional transformation.

---

## Consequences

### Positive

* Direct phenotype representation without decoding overhead
* Smooth continuous space, compatible with Gaussian mutation
* GA and RL optimize over the same parametric space $\theta \in \mathbb{R}^d$, making the optimization method the only difference
* Simple and stable operators
* Direct comparability with DQN and PPO guaranteed

### Negative

* Structure is not evolved, only weights
* High-dimensional space can cause genetic drift
* Sensitivity to weight scale
* Possible stagnation in flat regions of the loss landscape
* Introducing recurrent memory or topology evolution would require complete redesign of the representation

### Impact on other components

The chromosome dimension depends entirely on the architecture defined in ADR 0002. Any change in the architecture directly changes $d$.

The computational cost per GA generation is proportional to:

$$\text{cost}*{\text{gen}} \propto N*{\text{pop}} \times d \times \bar{T}_{\text{ep}}$$

The continuous representation avoids additional decoding overhead, keeping the cost per evaluation as low as possible within the $2\text{M}$ step budget.
