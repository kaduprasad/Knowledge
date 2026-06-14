# Deep Reinforcement Learning — Session 2 Exam Summary
**Course:** S2-25_AIMLCZG512 · **Instructor:** Prof. Sangeetha Viswanathan
**Topics covered:** Elements of RL · Value Functions · Tic-Tac-Toe RL · Exploration vs Exploitation · k-Armed Bandit · Action-Selection Strategies (Greedy, ε-Greedy)

---

## 1. When to apply Reinforcement Learning

A problem qualifies as RL when **all** of the following hold:
- **No pre-loaded dataset** — the agent must *interact* with the environment to gather data.
- The problem is **sequential decision-making** — one action changes the environment.
- There is a **defined goal** the agent must reach through experience.

> Distinguishes RL from supervised (labeled data) and unsupervised (pattern discovery) learning.

---

## 2. Core Elements of RL (quick recap)

| Element | Meaning |
|---|---|
| **Agent** | Learner / decision maker (e.g., robot, car, player) |
| **Environment** | Everything outside the agent's control |
| **State (S)** | Current situation / snapshot of the environment |
| **Action (A)** | Choice the agent makes in a state |
| **Reward (R)** | Immediate scalar feedback for an action (can be +ve, −ve, immediate, delayed, noisy) |
| **Policy (π)** | Strategy mapping state → action (can be **deterministic** or **stochastic**) |
| **Value Function** | Long-term expected reward of a state/action |
| **Model** *(optional)* | Agent's representation of the environment dynamics |

**Deterministic vs Stochastic Policy**
- *Deterministic:* From S1, action A → definitely lands on S2.
- *Stochastic:* From S1, action A → 20% chance S2, 80% chance S3.
- In dynamic environments, the final converged policy will typically be **stochastic** (but often almost-deterministic, e.g., 99% / 1%).

---

## 3. Reward vs Value Function ⭐ (high-yield exam topic)

| Aspect | **Reward** | **Value Function** |
|---|---|---|
| Nature | Immediate feedback signal | Long-term expected return |
| Time horizon | One time-step | Future rewards captured |
| Use | Tells *how good was this move now* | Tells *how good is it to be in this state / take this action* |

**Chess example given in class:**
- Rewards: win = **+1**, loss = **−1**, all intermediate moves = **0** → reward is *deferred / delayed*.
- Two intermediate states S2, S3 both yield reward 0, but the value function may say V(S2)=7, V(S3)=3 because S3 leads to losing the queen later.
- Hence **value function encodes future reward direction** while immediate reward cannot distinguish the two moves.

### Two kinds of value functions

$$
\boxed{V(s) = \text{State Value Function} = \mathbb{E}[\text{future rewards} \mid s]}
$$

$$
\boxed{Q(s, a) = \text{Action Value Function} = \mathbb{E}[\text{future rewards} \mid s, a]}
$$

> Policy and value function are a **close-knit family** — policy uses value functions to pick the best plan; the optimal policy is the chain of states with the greatest values.

---

## 4. Tic-Tac-Toe as an RL Problem

- **Zero-sum game:** one player wins (+1), the other loses (−1) → sum = 0.
- **Initial value of a state = 0.5** (both players equally likely to win at the start).
- RL learns by playing many games and **updating state values** based on outcomes.

### Underlying learning update (TD-style intuition)

$$
\boxed{V(s) \leftarrow V(s) + \alpha \big[ V(s') - V(s) \big]}
$$

where:
- $V(s)$ = current state value
- $V(s')$ = value of the next state
- $\alpha$ = **learning rate** (step-size)

### Effect of the learning rate α
| α value | Effect |
|---|---|
| **α = 0** | No update → weights/values never change → **no learning, no convergence** |
| **α = constant** | Fixed-rate learning — common hyperparameter setting |
| **α gradually decreased (but never 0)** | **Adaptive tuning** — model fine-tunes as it gains experience (preferred for convergence) |

---

## 5. Exploration vs Exploitation Trade-off ⭐⭐

- **Exploitation:** Always pick the action currently believed best (greedy).
- **Exploration:** Try other actions to discover potentially better ones.
- **Pure exploitation → may converge to a local best; pure exploration → may never converge.** Hence we **mix** them.

> "You are leaving room for mistakes — that room is your exploration." This is how the agent learns.

---

## 6. The k-Armed Bandit Problem

- Setup: **single state**, **k possible actions** (think k slot-machine levers).
- Goal: pick the **one action** that maximises expected reward.
- Assumption (for this module): **stationary reward distribution** — reward for each arm is drawn from a fixed probability distribution that doesn't change over time.
- Real world is typically **non-stationary** — distribution shifts over time.

### Action-Value Estimate Q(a) ⭐ (formula must-know)

$$
\boxed{\,Q_t(a) \;=\; \dfrac{\text{sum of rewards obtained when action } a \text{ was chosen}}{\text{number of times action } a \text{ was chosen}}\,}
$$

equivalently:

$$
Q_t(a) \;=\; \dfrac{\sum_{i=1}^{t-1} R_i \cdot \mathbb{1}(A_i = a)}{\sum_{i=1}^{t-1} \mathbb{1}(A_i = a)}
$$

- $Q^{*}(a)$ = **true value** of action *a* (unknown).
- $Q_t(a)$ = our **estimate** at time *t*. With repeated trials (under stationarity), $Q_t(a) \to Q^{*}(a)$ (Law of Large Numbers idea).

**Worked example from class** (T1…T5):

| Time | Action | Reward |
|---|---|---|
| T1 | A1 | 2 |
| T2 | A2 | 3 |
| T3 | A2 | 3 |
| T4 | A3 | 1 |
| T5 | A4 | 1 |

| Action | Q(a) | How |
|---|---|---|
| A1 | **2.0** | 2 / 1 |
| A2 | **3.0** | (3+3) / 2 |
| A3 | **1.0** | 1 / 1 |
| A4 | **1.0** | 1 / 1 |

> Note (clarified in class): A2's average is **3**, not 2.5 — the "2.5" stated mid-lecture was corrected to **3**.

---

## 7. Action-Selection Strategies

Four strategies in this module (only the **first two** are covered in this session; UCB and softmax come later):

1. **Greedy** ⭐
2. **ε-Greedy** ⭐
3. **UCB** (Upper Confidence Bound) — next session
4. **Softmax / Gradient-bandit** — next session

### 7.1 Greedy Action Selection

$$
\boxed{\,A_t \;=\; \arg\max_{a} \; Q_t(a)\,}
$$

- Always pick the action with the **highest current estimate**.
- **Problem:** Once a sub-optimal action looks best (e.g., due to a noisy early reward) it can be locked in forever → **never-ending loop** of exploitation.
- *Food-stall analogy:* Arjun tries 10 stalls once each, then on day 11 onward always returns to the one with the best first impression — but biryani might have just been lucky on day 1.

### 7.2 ε-Greedy Action Selection ⭐

$$
\boxed{
A_t =
\begin{cases}
\arg\max_{a} Q_t(a) & \text{with probability } 1-\varepsilon \quad \text{(exploit)} \\
\text{random action} & \text{with probability } \varepsilon \quad \text{(explore)}
\end{cases}
}
$$

- ε is a **hyperparameter** (like learning rate).
- Example: ε = 0.2 → 20% explore, 80% exploit.
- During exploration the action is **uniformly random** among *all* actions (the greedy action itself can also be re-picked).

#### Pseudocode (as shown in class)
```text
Set ε = 0.05
For each time step t:
    Generate random number p ∈ [0, 1]
    If p > ε:                         # 1 − ε  → exploit
        A_t = argmax Q(a)
    Else:                             # ε      → explore
        A_t = random action from all actions
    Observe reward R_t
    Update Q(A_t) using the running average formula
```

---

## 8. Classic Numerical: Probability of Selecting the Greedy Action under ε-Greedy ⭐⭐ (likely exam Q)

> *"With 2 actions and ε = 0.5, what is the probability the greedy action is chosen?"*

Reasoning:

1. With probability $1-\varepsilon = 0.5$ → exploit → greedy action chosen for sure.
   - Contribution: $0.5 \times 1 = 0.5$
2. With probability $\varepsilon = 0.5$ → explore → uniformly random over 2 actions → greedy action still has $\tfrac{1}{2}$ chance.
   - Contribution: $0.5 \times 0.5 = 0.25$

$$
\boxed{\,P(\text{greedy}) \;=\; (1-\varepsilon) \;+\; \varepsilon \cdot \tfrac{1}{|\mathcal{A}|}
\;=\; 0.5 + 0.25 \;=\; \mathbf{0.75}\,}
$$

**Generalised formula** for k actions:

$$
P(\text{greedy action}) \;=\; (1-\varepsilon) \;+\; \dfrac{\varepsilon}{k}
$$

$$
P(\text{any specific non-greedy action}) \;=\; \dfrac{\varepsilon}{k}
$$

---

## 9. Stationary vs Non-Stationary Reward Distributions

- **Stationary:** reward distribution is fixed over time (e.g., always drawn from range [10, 100]).
  → Sample-mean Q(a) converges to true Q*(a).
- **Non-stationary:** distribution shifts (e.g., casino owner changes payouts weekly).
  → Sample mean is **biased**; we must weight recent rewards more (covered later via exponential / constant-step-size updates).

**Why use the average even though it's noisy?** Reality has unknown state changes and noisy rewards; *under stationarity*, averaging has been shown empirically/theoretically to converge to the true action value.

---

## 10. Key Insights & Likely Exam Pointers

- "**Domain expert** defines the reward function and goal; the **learning agent** discovers the policy." — multiple students confirmed this.
- **At least once each action must be tried** before action-value estimates become meaningful; otherwise Q(a) stays 0 forever.
- **ε and α are hyperparameters** — tune them; both can be decayed.
- Reward = **immediate**; Value = **long-term** (chess queen-loss example).
- Tic-tac-toe is a **zero-sum** game; initial value 0.5.
- **Greedy = exploit only; ε-greedy = mix exploit + explore.**
- **Multi-agent RL** is a separate field (out of scope here).

---

## 11. Formula Cheat-Sheet (one glance)

| # | Formula | Meaning |
|---|---|---|
| 1 | $\,Q_t(a)=\dfrac{\sum R_i \cdot \mathbb{1}(A_i=a)}{\sum \mathbb{1}(A_i=a)}\,$ | Sample-average action value estimate |
| 2 | $\,A_t = \arg\max_a Q_t(a)\,$ | Greedy selection |
| 3 | $\,A_t = \begin{cases}\arg\max_a Q_t(a) & \text{w.p. }1-\varepsilon\\ \text{random} & \text{w.p. }\varepsilon\end{cases}\,$ | ε-Greedy selection |
| 4 | $\,P(\text{greedy})=(1-\varepsilon)+\dfrac{\varepsilon}{k}\,$ | Probability greedy action is chosen under ε-greedy |
| 5 | $\,V(s) \leftarrow V(s)+\alpha\,[V(s')-V(s)]\,$ | TD value-update (tic-tac-toe style) |
| 6 | $\,V(s)=\mathbb{E}[\text{future rewards}\mid s]\,$ | State value |
| 7 | $\,Q(s,a)=\mathbb{E}[\text{future rewards}\mid s,a]\,$ | Action value |

---

*End of Session 2 summary — next session continues with UCB action-selection strategy and revisits the ε-greedy probability question.*
