# Deep Reinforcement Learning — Session 5 Exam Summary
**Date:** May 23, 2026 · **Instructor:** Prof. Sangeetha Viswanathan
**Topics:** Goals & Rewards · Episodic vs Continuing · **Discounted Return** · Cart-Pole · **Policy** $\pi$ · **State-value $V_\pi(s)$ vs Action-value $Q_\pi(s,a)$** · **Bellman Expectation Equation** (bootstrapping) · Race-Car & Grid examples · **Optimal Policy & Bellman Optimality** · Intro to Dynamic Programming

---

## 1. The Reward Hypothesis

> "Goals and purposes can be thought of as the maximisation of the expected cumulative reward."

- Reward function tells **WHAT** to achieve, **NEVER HOW** to achieve it.
- The "how" is discovered by the learning algorithm via interaction.
- A *first-order* Markov framework is the default. (Higher-order frameworks exist — $n$-th order — but are out of scope.)

---

## 2. Episodic vs Continuing Tasks ⭐

| Type | Has terminal state? | Example |
|---|---|---|
| **Episodic** | Yes (clear end) | Game of chess, single round of Tic-tac-toe |
| **Continuing** | No defined end | Autonomous car, lifelong robot operation, "human learning" |

- Each finite interaction sequence in an episodic task = one **episode** / **trajectory**.
- Cart-pole can be modelled as either: it's natural continuing, but capped to "episode ends when pole tilts beyond a threshold" makes it episodic.

---

## 3. Return $G_t$ — Cumulative Reward from $t$

### Episodic, undiscounted (finite horizon $T$):
$$
G_t = R_{t+1} + R_{t+2} + \dots + R_T
$$

### Discounted return ⭐⭐⭐ (universal definition)

$$
\boxed{\,G_t \;=\; R_{t+1} + \gamma R_{t+2} + \gamma^2 R_{t+3} + \dots \;=\; \sum_{k=0}^{\infty} \gamma^{k}\, R_{t+k+1}\,}
$$

Recursive form (very important for Bellman):

$$
\boxed{\,G_t \;=\; R_{t+1} + \gamma\, G_{t+1}\,}
$$

where $\gamma \in [0, 1]$ is the **discount factor / discount rate** — a hyper-parameter.

### Why discount?
- Future rewards are uncertain.
- Prevents an agent from being "blindsided" by far-off rewards in stochastic worlds.
- Keeps the sum finite for continuing tasks.

### Edge cases of γ

| γ | Behaviour |
|---|---|
| **γ = 0** | Myopic / greedy — only the immediate reward $R_{t+1}$ matters |
| **γ = 1** | All rewards weighted equally (no discount); only valid for episodic tasks |
| **0 < γ < 1** | Standard; recent rewards weigh more, distant rewards decay geometrically |

### Constant-Reward Simplification ⭐ (exam shortcut)

If every reward is the same constant $R$:

$$
\boxed{\,G_t = \dfrac{R}{1 - \gamma}\,}
$$

### Worked numerical (asked in class)
Given $R_2 = 2$, $R_3 = R_4 = \dots = 7$, $\gamma = 0.1$. Find $G_1, G_2$.

- $G_2 = \dfrac{7}{1-0.1} = \dfrac{7}{0.9}$
- $G_1 = R_2 + \gamma\, G_2 = 2 + 0.1 \cdot \dfrac{7}{0.9}$

> Episodic tasks technically don't need discounting, but practitioners almost always include γ to **shape behaviour** (e.g., encourage faster wins in chess).

---

## 4. Policy $\pi$ ⭐

$$
\boxed{\,\pi(a \mid s) \;=\; \Pr\{A_t = a \mid S_t = s\}\,}
$$

- A **mapping from states to actions** (or distribution over actions).
- **Deterministic policy:** $\pi(s) = a$ (one action chosen with certainty).
- **Stochastic policy:** probability distribution over $\mathcal{A}$.
- Goal of RL = find the **optimal policy** $\pi^{*}$.

---

## 5. Value Functions ⭐⭐⭐

### (a) State-Value Function $V_\pi(s)$
"How good is it to be in state $s$ while following policy $\pi$?"

$$
\boxed{\,V_\pi(s) \;=\; \mathbb{E}_\pi\!\left[\,G_t \mid S_t = s\,\right] \;=\; \mathbb{E}_\pi\!\left[\,\sum_{k=0}^{\infty}\gamma^{k}R_{t+k+1}\,\Big|\,S_t = s\right]\,}
$$

### (b) Action-Value Function $Q_\pi(s, a)$
"How good is it to take action $a$ from state $s$ then follow $\pi$?"

$$
\boxed{\,Q_\pi(s, a) \;=\; \mathbb{E}_\pi\!\left[\,G_t \mid S_t = s,\, A_t = a\,\right]\,}
$$

### Relationship
$$
\boxed{\,V_\pi(s) \;=\; \sum_a \pi(a\mid s)\, Q_\pi(s, a)\,}
$$

> **Insight:** the state-value $V_\pi$ **contains** the action-value $Q_\pi$ inside (marginalised over the policy). Some algorithms work directly with $Q$ (e.g., Q-learning); others with $V$ (e.g., value iteration). Both co-exist and call each other recursively.

### Difference from earlier "action-value estimate" of MAB
- MAB $Q_t(a)$ = plain **sample average** of rewards (no future, no state).
- Full-RL $Q_\pi(s,a)$ = **expected discounted return** (includes future rewards & state transitions).

---

## 6. Bellman Expectation Equations ⭐⭐⭐ (most important of the session)

### For state-value

$$
\boxed{\,V_\pi(s) \;=\; \sum_a \pi(a\mid s) \sum_{s', r} p(s', r\mid s, a)\,\Bigl[\,r + \gamma\, V_\pi(s')\,\Bigr]\,}
$$

### For action-value

$$
\boxed{\,Q_\pi(s, a) \;=\; \sum_{s', r} p(s', r\mid s, a)\,\Bigl[\,r + \gamma\, \sum_{a'} \pi(a'\mid s')\, Q_\pi(s', a')\,\Bigr]\,}
$$

> Both are **recursive**: value of current state expressed in terms of value of the next state → **bootstrapping** (a concept introduced by Richard **Bellman**).

### Plain-English meaning
$$
\text{Value}(s) \;=\; \text{Immediate reward} \;+\; \gamma \cdot \text{Discounted value of the next state}
$$

---

## 7. Grid-World Verification Example (from class)

Given an equiprobable random policy $\pi(a\mid s) = 0.25$ (4 actions: N, E, S, W) in a **deterministic** environment ($P=1$ for the intended cell):

For a state cell whose neighbours have values $\{2.3, 0.4, -0.4, 0.7\}$ and step reward = 0 with $\gamma = 0.9$:

$$
V(s) = 0.25\,(0 + 0.9 \cdot 2.3) + 0.25\,(0 + 0.9 \cdot 0.4) + 0.25\,(0 + 0.9\cdot(-0.4)) + 0.25\,(0 + 0.9\cdot 0.7) \approx 0.7
$$

For the **action-value** of "down" from the same state (to a neighbour with value $-1.2$):

$$
Q(s,\text{down}) = 1 \cdot \bigl(0 + 0.9 \cdot (-1.2)\bigr) = -1.08
$$

> When asking $Q(s,a)$ you **don't** multiply by $\pi(a\mid s)$ — the action is already fixed.

For a **stochastic** version: with 20% intended / 80% other slip, replace the 1 with the actual transition probability and sum over all reached states.

---

## 8. Race-Car Example (recurring)

- **States:** {Cool, Warm, Overheated (terminal)}
- **Actions:** {Slow, Fast}
- Stochastic transitions (e.g., from *Warm* + *Slow*: 0.5 → Warm, 0.5 → Cool)
- Rewards: $+1$ for Slow, $+2$ for Fast, $-10$ for Overheated.

This will be solved fully in Session 6 using Dynamic Programming.

---

## 9. Optimal Policy and Optimal Value Functions ⭐⭐

### Definition of optimality

$$
\boxed{\,\pi \geq \pi' \iff V_\pi(s) \geq V_{\pi'}(s)\quad \forall s\in\mathcal{S}\,}
$$

The **optimal policy** $\pi^{*}$ achieves the maximum value in every state. There can be **multiple** optimal policies (all yielding the same maximum $V$ values).

### Optimal value functions

$$
\boxed{\,V^{*}(s) \;=\; \max_\pi V_\pi(s)\,}
\qquad
\boxed{\,Q^{*}(s, a) \;=\; \max_\pi Q_\pi(s, a)\,}
$$

### Bellman Optimality Equations ⭐⭐ (replace $\sum$ over actions with $\max$)

$$
\boxed{\,V^{*}(s) \;=\; \max_{a}\,\sum_{s', r} p(s', r\mid s, a)\,\Bigl[\,r + \gamma\, V^{*}(s')\,\Bigr]\,}
$$

$$
\boxed{\,Q^{*}(s, a) \;=\; \sum_{s', r} p(s', r\mid s, a)\,\Bigl[\,r + \gamma\,\max_{a'} Q^{*}(s', a')\,\Bigr]\,}
$$

> Key change from expectation to optimality: **replace summation over actions with the `max` operator**.

### Greedy policy from $Q^{*}$

$$
\pi^{*}(s) \;=\; \arg\max_a Q^{*}(s, a)
$$

---

## 10. Bootstrapping (Bellman's Core Idea)

- "Estimate the current state using the (estimated) value of the next state."
- Avoids the intractable alternative of summing returns along *every possible trajectory*.
- Forms the basis of **Dynamic Programming**, **TD-learning**, and **Q-learning**.

---

## 11. Stopping / Convergence

- Iterate Bellman updates until $\bigl|V_{k+1}(s) - V_k(s)\bigr| < \epsilon$ for all $s$ → **policy converged**.
- Detailed convergence proof and algorithm (value iteration / policy iteration) come in Session 6 (DP).

---

## 12. Important Tidbits & Likely Exam Pointers

- **State-value alone is not enough** for action selection without the model; **action-value** lets you directly pick $\arg\max_a Q(s,a)$.
- Optimality: replace $\sum_a \pi(a|s)$ with $\max_a$.
- Multiple optimal policies are possible because reward landscapes can have ties; algorithms can pick any of them.
- For continuing tasks, **discounting is mandatory**; for episodic tasks, it is optional but standard practice.
- Course will focus on **episodic tasks** going forward (continuing is computationally hard to trace).

---

## 13. Formula Cheat-Sheet

| # | Formula | Meaning |
|---|---|---|
| 1 | $G_t = R_{t+1}+\gamma R_{t+2}+\dots = \sum_k \gamma^k R_{t+k+1}$ | Discounted return |
| 2 | $G_t = R_{t+1}+\gamma G_{t+1}$ | Recursive return |
| 3 | $G_t = R/(1-\gamma)$ | Constant reward shortcut |
| 4 | $\pi(a\mid s)=\Pr\{A_t=a\mid S_t=s\}$ | Policy |
| 5 | $V_\pi(s)=\mathbb{E}_\pi[G_t\mid s]$ | State-value |
| 6 | $Q_\pi(s,a)=\mathbb{E}_\pi[G_t\mid s,a]$ | Action-value |
| 7 | $V_\pi(s)=\sum_a \pi(a\mid s)\,Q_\pi(s,a)$ | $V$ from $Q$ |
| 8 | $V_\pi(s)=\sum_a\pi(a\mid s)\sum_{s',r}p(s',r\mid s,a)[r+\gamma V_\pi(s')]$ | **Bellman expectation – V** |
| 9 | $Q_\pi(s,a)=\sum_{s',r}p(s',r\mid s,a)[r+\gamma\sum_{a'}\pi(a'\mid s')Q_\pi(s',a')]$ | **Bellman expectation – Q** |
| 10 | $V^{*}(s)=\max_a \sum_{s',r}p(s',r\mid s,a)[r+\gamma V^{*}(s')]$ | **Bellman optimality – V** |
| 11 | $Q^{*}(s,a)=\sum_{s',r}p(s',r\mid s,a)[r+\gamma\max_{a'}Q^{*}(s',a')]$ | **Bellman optimality – Q** |
| 12 | $\pi^{*}(s)=\arg\max_a Q^{*}(s,a)$ | Greedy optimal policy |

---

*End of Session 5 summary — Session 6 begins Dynamic Programming (value iteration / policy iteration) on the race-car MDP.*
