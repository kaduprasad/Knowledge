# Deep Reinforcement Learning — Session 4 Exam Summary
**Date:** May 16, 2026 · **Instructor:** Prof. Sangeetha Viswanathan
**Topics:** Non-stationary update numerical · MAB → Contextual Bandits → Full RL · **Markov Decision Process (MDP)** · MDP 5-tuple · **Model Dynamics, State-Transition Probability, Expected Reward** · Grid World · Agent–Environment boundary · Recycling-Robot transition diagram

---

## 1. Clarification on Optimistic Initial Values (carried from Session 3)

- The Sutton-Barto optimistic-initial-value experiment used **constant α = 0.1** even in a **stationary** setup.
- It is **legal** to use a constant step-size in a stationary problem; but for true non-stationary, constant α is **required**.

---

## 2. Non-Stationary Update — Worked Numerical ⭐

**Given:** rewards $R_1=10,\ R_2=0,\ R_3=\dots,\ Q_1=0,\ \alpha = 1/2$.

Update: $Q_{n+1} = Q_n + \alpha(R_n - Q_n)$

| Step | Computation | Result |
|---|---|---|
| $Q_2$ | $0 + 0.5\,(10 - 0)$ | **5** |
| $Q_3$ | $5 + 0.5\,(0 - 5)$ | **2.5** |
| $Q_4$ | $2.5 + 0.5\,(R_3 - 2.5)$ | depends on $R_3$ |

### Importance weights (decaying)
For $\alpha = 0.5$, weights on past rewards from most recent → oldest:
$$
0.5,\; 0.25,\; 0.125,\; 0.0625, \dots
$$
Each one is half the previous → **recent rewards matter more**, distant past is forgotten.

### Why $1/n$ for stationary but $\alpha$ for non-stationary

| | Stationary | Non-stationary |
|---|---|---|
| Step size | $\dfrac{1}{n}$ shrinks → small at large $n$ | $\alpha$ fixed |
| Effect | New samples barely move Q once well-estimated | Stays responsive to drift |
| Convergence | $Q_t(a) \to Q^{*}(a)$ | Tracks current distribution |

---

## 3. From MAB → Contextual Bandit → Full RL

| Setup | Tuple | What's missing |
|---|---|---|
| **MAB (k-armed bandit)** | $(A, R)$ | No state, no transitions |
| **Contextual Bandit** | $(S, A, R)$ | Has context (state) **but** no state transitions — single-step decision per context |
| **Full RL / MDP** | $(S, A, P, R, \gamma)$ | Includes transitions $S \to S'$ — sequential decisions |

**Example (Arjun):**
- **MAB:** "Which food stall gives max reward?" — no context, no state changes.
- **Contextual:** "Given user's age group / context $X$, which ad/product to push?"
- **Full RL:** "Arjun was healthy → drank too-sweet juice → fell sick" — state changes matter.

---

## 4. Markov Property ⭐ (foundational)

$$
\boxed{\,P(S_{t+1}\mid S_t, A_t) = P(S_{t+1}\mid S_t, A_t, S_{t-1}, A_{t-1}, \dots, S_0, A_0)\,}
$$

> "The future depends only on the present state, not on the entire past history."

Knowing $S_t$ alone is enough to predict $S_{t+1}$ — all relevant past info is summarised in $S_t$.

**Chess intuition:** Showing the *current board* alone is enough to continue the game — no need to know previous moves.

---

## 5. Agent–Environment Interaction Diagram

```
       ┌────────────┐   A_t
       │   Agent    │ ─────────►
       └────▲───────┘
            │ R_{t+1},  S_{t+1}
       ┌────┴───────┐
       │Environment │ ◄─────────
       └────────────┘
```

- Reward and next state are **one step ahead** ($R_{t+1}, S_{t+1}$) because the environment must react first.

### SAR Sequence / Trajectory / Episode

$$
S_0, A_0, R_1, S_1, A_1, R_2, S_2, A_2, R_3, \dots
$$

A complete interaction is called a **trajectory** or **episode**.

---

## 6. MDP — The 5-Tuple Vector ⭐⭐ (must-know for exam)

$$
\boxed{\,\text{MDP} = \langle\, \mathcal{S},\; \mathcal{A},\; \mathcal{P},\; \mathcal{R},\; \text{(start / terminal)}\,\rangle\,}
$$

| Symbol | Meaning |
|---|---|
| $\mathcal{S}$ | Set of all possible states |
| $\mathcal{A}$ | Set of all possible actions |
| $\mathcal{P}$ | **State-transition probability** $P(s'\mid s,a)$ |
| $\mathcal{R}$ | **Reward function** $R(s,a,s')$ |
| Start / Terminal | Start state(s) and terminal/absorbing state(s) |

> Sometimes written as a 5-tuple $(S, A, P, R, \gamma)$ where $\gamma$ replaces start/terminal info.

### Grid-World Example (3 × 4, given in class)
- $\mathcal{S} = \{(1,1), (1,2), \dots, (4,3)\}$ minus the blocked cell $(2,2)$.
- $\mathcal{A} = \{N, E, W\}$ *(South not allowed)*
- $\mathcal{P}$ (slippery floor):
  - $P(\text{intended}\mid s, N)=0.8$
  - $P(\text{slip W}\mid s, N)=0.1$
  - $P(\text{slip E}\mid s, N)=0.1$
- $\mathcal{R}$:
  - $-0.1$ for every step
  - $+1$ at $(4,3)$
  - $-1$ at $(4,2)$
- Start = $(1,3)$, Terminal = $(4,3)$

---

## 7. The Three Fundamental MDP Equations ⭐⭐⭐ (very high-yield)

### (a) Model Dynamics — complete one-step description

$$
\boxed{\,p(s', r \mid s, a) \;=\; \Pr\{\,S_{t+1}=s',\, R_{t+1}=r \mid S_t=s,\, A_t=a\,\}\,}
$$

> "Given I am in state $s$ and take action $a$, what is the probability of landing in $s'$ AND simultaneously getting reward $r$?"
> Contains everything needed to describe a one-step interaction.

### (b) State-Transition Probability — marginalised over rewards

$$
\boxed{\,p(s' \mid s, a) \;=\; \sum_{r \in \mathcal{R}} p(s', r \mid s, a)\,}
$$

> "Where do I end up next?" — sums over all possible rewards.
> Multiple rewards are possible for the same $(s, a, s')$ triple (e.g. reach-customer-fast vs reach-customer-slow).

### (c) Expected Reward for $(s, a, s')$ Triple

$$
\boxed{\,r(s, a, s') \;=\; \mathbb{E}[\,R_{t+1}\mid S_t=s,\, A_t=a,\, S_{t+1}=s'\,] \;=\; \sum_{r}\, r\,\dfrac{p(s', r\mid s, a)}{p(s' \mid s, a)}\,}
$$

> Average reward observed on a specific transition. Useful for algorithms that need expected-reward bookkeeping.

### Easier form: Expected reward for $(s, a)$ pair
$$
r(s, a) \;=\; \mathbb{E}[\,R_{t+1}\mid s, a\,] \;=\; \sum_{r}\, r\,\sum_{s'} p(s', r\mid s, a)
$$

---

## 8. Why Multiple Rewards for the Same (s, a, s')?

**Warehouse robot example** (from class):
- State: $s$ = robot at start. Action: $a$ = move forward.
- Same next state $s'$ = "reach customer" can come with **different rewards**:
  - $+10$ → reach customer **fast**
  - $+5$ → reach customer **slow** (took longer)
- So $p(s', r\mid s, a)$ needs both $s'$ and $r$ axes.

---

## 9. Agent–Environment Boundary is **Abstract and Flexible**

| Component | Granularity examples |
|---|---|
| **Time step** | 1 ms (motor control), 1 chess move, 1 week (stock-trading) |
| **Action** | Low-level (apply 3.4 V to motor, 10 N brake) **or** high-level (voice command, "buy stock") |
| **State** | Raw sensor readings (camera pixels) **or** abstract labels (high-energy / low-energy, full / half / empty battery) |

> A "state" is the **situation of the agent in its environment**, not just the environment itself.

---

## 10. Recycling-Robot Example (Sutton & Barto, illustrative only)

- **States:** $\{\text{high}, \text{low}\}$ (battery level)
- **Actions:** high → {search, wait}; low → {search, wait, recharge}
- **Rewards:**
  - $r_{\text{search}}$ if searching
  - $r_{\text{wait}}$ if waiting ($r_{\text{search}} > r_{\text{wait}}$)
  - $-3$ (large penalty) if battery dies during search
  - $0$ for recharge
- **Transitions** (parameters $\alpha, \beta$):

| Current | Action | Next | Prob | Reward |
|---|---|---|---|---|
| high | search | high | $\alpha$ | $r_{\text{search}}$ |
| high | search | low | $1-\alpha$ | $r_{\text{search}}$ |
| high | wait | high | 1 | $r_{\text{wait}}$ |
| low | search | low | $\beta$ | $r_{\text{search}}$ |
| low | search | high | $1-\beta$ | $-3$ (rescue + recharge) |
| low | wait | low | 1 | $r_{\text{wait}}$ |
| low | recharge | high | 1 | $0$ |

> Skill: be able to **convert a transition diagram → MDP table** and vice-versa.

---

## 11. Important Notes & Exam Pointers

- Be ready to **write the MDP formulation** ($S, A, P, R$, start, terminal) for any described scenario — this is a graded exam skill.
- MDP is a **framework**, not an algorithm. RL algorithms (DP, MC, TD, Q-learning, …) operate **on** an MDP.
- Course starts with **model-known** settings (you have $P, R$); model-free RL covered later (Session 6+).
- The state $s$ should capture **everything relevant** so the Markov property holds — design $s$ carefully.

---

## 12. Formula Cheat-Sheet

| # | Formula | Meaning |
|---|---|---|
| 1 | $Q_{n+1}=Q_n+\alpha(R_n-Q_n)$ | Non-stationary incremental update |
| 2 | Weight of $R_{n-k}$ = $\alpha(1-\alpha)^{k}$ | Exponential recency weighting |
| 3 | $P(S_{t+1}\mid S_t,A_t)=P(S_{t+1}\mid \text{full history})$ | **Markov property** |
| 4 | $p(s',r\mid s,a)$ | **Model dynamics** (one-step joint) |
| 5 | $p(s'\mid s,a)=\sum_r p(s',r\mid s,a)$ | **State-transition probability** |
| 6 | $r(s,a,s')=\sum_r r\,\dfrac{p(s',r\mid s,a)}{p(s'\mid s,a)}$ | **Expected reward** for triple |
| 7 | $r(s,a)=\sum_r r\sum_{s'} p(s',r\mid s,a)$ | Expected reward for $(s,a)$ |
| 8 | MDP $= \langle S, A, P, R, \text{(start/terminal)}\rangle$ | **5-tuple** representation |

---

*End of Session 4 summary — Session 5 introduces goals, returns, value functions, and Bellman equations.*
