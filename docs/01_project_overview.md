# 01 — Project Overview

## What it is

This project implements and compares four optimization paradigms —random baseline, evolutionary algorithms (GA), value-based learning (DQN), and policy optimization (PPO)— in a dynamic survival environment with real-time rewards.

The objective is to analyze how they differ in behavior, stability, and learning dynamics when facing the same problem under structurally controlled conditions.

It is a comparative experimental study designed as an advanced portfolio project and a reproducible academic experiment. It is not a visual demo of an agent playing, but rather a methodological comparison between paradigms.

---

## Why it exists

In the literature and in practice, it is often claimed that:

* "Evolutionary algorithms are robust to noise"
* "Modern RL converges faster"
* "Gradient-based methods outperform evolutionary ones"

These claims are rarely verified under structural equality: same environment, same architecture, same interaction budget, same seeds.

This project closes that gap by showing in a controlled way how these approaches actually behave when competing under equivalent conditions.

The Flappy Bird–type environment was chosen because it is dynamic, sequential, with simple discrete action and dense reward. It allows many fast iterations and is complex enough to require real learning, without introducing unnecessary high-dimensional visual noise. It is structurally simple, but not behaviorally trivial.

---

## What it produces

* Implemented and documented dynamic environment
* Implementations of Random, GA, DQN, and PPO
* Structured logging system by experiment
* Reproducible scripts with seed control
* Visualizations of learning curves and emergent behavior
* Comparative technical report with statistical analysis
* Dockerfile for full reproducibility

It is not just code. It is a complete experimental system that someone can clone, run, and extend.

---

## Scope

Within the project:

* Controlled comparison of the four methods from scratch
* Comparable network architecture in number of parameters
* Systematic evaluation across multiple seeds
* Interpretive analysis of results

Explicitly out of scope:

* Exhaustive search for optimal hyperparameters
* Comparison across multiple environments
* High-dimensional visual input (CNN)
* Advanced variants (NEAT, SAC, Rainbow, etc.)
* Multi-agent
* Formal theoretical convergence analysis

The focus is conceptual clarity under controlled conditions, not breadth.

---

## What it is not

* It is not a universal benchmark of RL or evolutionary algorithms
* It is not a state-of-the-art implementation
* It does not claim that its results generalize to all environments
* It does not aim to demonstrate that one paradigm is universally superior

If PPO obtains better results here, the correct statement is: "PPO converged faster in this environment under this budget," not "PPO is always better."

---

## Demonstrated technical value

**Experimental design**: methodological control, not just implementation. The comparison conditions are defined before running any experiment.

**Understanding of multiple paradigms**: evolution, value-based RL, and policy optimization implemented from their foundations.

**Systems engineering**: modular architecture with clear separation between environment, agent, and model. Structured logging. Full reproducibility.

**Analytical capability**: interpretation of learning curves, variance analysis across runs, critical discussion of tradeoffs between methods.

**Conceptual maturity**: comparing algorithms is not running code and looking at the score. It is designing controlled conditions, defining metrics before experimenting, and analyzing results with statistical rigor.
