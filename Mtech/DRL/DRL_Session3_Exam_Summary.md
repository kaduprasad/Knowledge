# Deep Reinforcement Learning — Session 3 Exam Summary
**Date:** May 9, 2026 · **Instructor:** Prof. Sangeetha Viswanathan
**Topics:** ε-Greedy probability (worked) · 10-armed test-bed · ε vs α trade-off · Bandit numerical (greedy vs exploration) · Incremental update · Non-stationary updates · Optimistic initial values · **UCB** · Softmax / policy preference

---

## 1. Quick Recap of k-Armed Bandit

- **Goal:** Pick one of $k$ actions repeatedly to maximise cumulative reward.
- **Action-value estimate (sample average):**

$$
\boxed{\,Q_t(a) = \dfrac{\text{sum of rewards when } a \text{ was chosen}}{\text{number of times } a \text{ was chosen}}\,}
$$

- **Greedy:** $A_t = \arg\max_a Q_t(a)$
- **ε-Greedy:** explore w.p. $\varepsilon$, exploit w.p. $1-\varepsilon$.

In simulation, a uniform random number $p \in [0,1]$ decides per trial:
- `if p < ε → explore` else `exploit`. Over many trials, the fraction explored ≈ ε.

---

## 2. ε-Greedy: Probability of Picking the Greedy Action ⭐⭐ (exam favourite)

$$
\boxed{\,P(\text{greedy}) = (1-\varepsilon) + \dfrac{\varepsilon}{k}\,}
$$

### Worked examples (asked in class)

| Scenario | Computation | Answer |
|---|---|---|
| ε = 0.5, k = 2 | $0.5 + 0.5/2$ | **0.75** |
| ε = 0.5, k = 4 | $0.5 + 0.5/4$ | **0.625** |
| ε = 0.1, k = 3 | $0.9 + 0.1/3 \approx 0.9 + 0.0333$ | **0.933** |

> Remember: even during the exploration branch, the greedy action can be re-picked uniformly with chance $1/k$.

---

## 3. 10-Armed Test-Bed Experiment (Sutton & Barto)

- 10 actions, 200 trials, rewards drawn from a Gaussian / Normal distribution.
- Plots compared **ε = 0**, **ε = 0.01**, **ε = 0.1**.

### Key inferences

| ε value | Behaviour |
|---|---|
| **ε = 0** (pure greedy) | Gets stuck on first decent action → low % optimal action |
| **ε = 0.01** | Slow but high final optimal-action % (converges very steadily) |
| **ε = 0.1** | Reaches ~80% optimal action fastest; commonly used |

> "Optimal action" ≠ "greedy action". Greedy picks max-current-estimate. Optimal is the action with truly max reward — only known with hindsight.

---

## 4. ε vs Learning Rate (α) Trade-off

Both are **independent hyperparameters**:

| ε | α | Effect |
|---|---|---|
| **High ε, Low α** | Lots of exploration, slow learning → **may diverge** |
| **Low ε, High α** | Little exploration, fast learning → traps to local best, **may collapse / unstable** |
| **High ε, High α** | Unstable — model collapses eventually |
| **Low ε, Low α** | Too slow and too narrow — bad |

**Rule of thumb:** controlled environment (e.g., robotic vacuum) → low ε is OK; uncertain environment → higher ε needed.

---

## 5. Bandit Numerical — Was the action Greedy or Exploration? ⭐⭐ (textbook Q)

> 4 actions, initial Q = 0 for all. Observed sequence:
> A₁=1, A₂=2, A₃=2, A₄=2, A₅=3 (reward for each step is the action number after first; here use sample-mean updates as shown in class).

### Step-by-step Q updates

| t | Action | Reward | Q(A1) | Q(A2) | Q(A3) | Q(A4) |
|---|---|---|---|---|---|---|
| 0 | — | — | 0 | 0 | 0 | 0 |
| 1 | A1 | 1 | **1** | 0 | 0 | 0 |
| 2 | A2 | 1 | 1 | **1** | 0 | 0 |
| 3 | A2 | 2 | 1 | **1.5** = (1+2)/2 | 0 | 0 |
| 4 | A2 | 2 | 1 | **1.67** ≈ 5/3 | 0 | 0 |
| 5 | A3 | 0 | 1 | 1.67 | **0** | 0 |

### Verdict per time-step (compare actual action vs argmax-Q from previous row)

| t | Action taken | Greedy was | Conclusion |
|---|---|---|---|
| 1 | A1 | A1 (all 0, tie) | **Greedy / Possibly Explore** |
| 2 | A2 | A1 *and* A2 tied at 1 | **Possibly explore OR exploit** |
| 3 | A2 | A2 (max=1) | **Greedy** |
| 4 | A2 | A2 (max=1.5) | **Greedy** |
| 5 | A3 | A2 (max=1.67) | **Exploration definitely occurred** |

> Trick: When current Q has a **unique max** and action ≠ that max → exploration *definitely* occurred.
> When a tie exists → exploration *might* have occurred.

---

## 6. Incremental Implementation of Q (Online update) ⭐ (must-know derivation)

Instead of recomputing the average from scratch each step:

$$
\boxed{\,Q_{n+1} = Q_n + \dfrac{1}{n}\bigl[R_n - Q_n\bigr]\,}
$$

Generic form:

$$
\boxed{\,\text{NewEstimate} \leftarrow \text{OldEstimate} + \text{StepSize}\,\bigl[\text{Target} - \text{OldEstimate}\bigr]\,}
$$

- **Target** = the reward just observed, $R_n$.
- **StepSize** = $\dfrac{1}{n}$ for **stationary** problems → sample mean.

> RL convention: action $A_t$ is taken at time $t$; the reward $R_{t+1}$ is observed at the *next* step because the environment transitions first.

---

## 7. Non-Stationary Updates — Constant Step-Size α ⭐

For changing reward distributions, replace $\dfrac{1}{n}$ with constant $\alpha \in (0, 1]$:

$$
\boxed{\,Q_{n+1} = Q_n + \alpha\bigl[R_n - Q_n\bigr]\,}
$$

### Equivalent exponentially-weighted form

$$
\boxed{\,Q_{n+1} = (1-\alpha)^{n}\,Q_1 \;+\; \sum_{i=1}^{n} \alpha(1-\alpha)^{n-i}\,R_i\,}
$$

- Weight of the *most recent* reward = $\alpha$.
- Weight of older rewards decays **exponentially** as $(1-\alpha)^{k}$.
- Distant past is effectively "forgotten" → ideal for non-stationary worlds where the reward distribution shifts over time.

| Setup | Step size | Why |
|---|---|---|
| Stationary | $1/n$ | Convergence to true $Q^{*}(a)$ (Law of Large Numbers) |
| Non-stationary | Constant $\alpha$ | Track current distribution; downweight stale samples |

---

## 8. Optimistic Initial Values

- Instead of $Q_1(a) = 0$, set $Q_1(a) = +5$ (or any high value).
- The agent finds every action **disappointing** at first → forced to **explore** all actions early.
- Experiment: stationary, 10-arm, **ε=0** with $Q_1=+5$ outperformed **ε=0.1** with $Q_1=0$ after ~1000 steps (~82% optimal vs ~65%).

**Limitations**
- Works only for **stationary** problems with **controlled / known** environments.
- Does **not** encourage continued exploration (only an early kick).
- Inappropriate when reward distribution changes over time.

---

## 9. Upper-Confidence-Bound (UCB) Action Selection ⭐⭐

Encourages trying **least-tried** actions whose value is most uncertain.

$$
\boxed{\,A_t \;=\; \arg\max_a \left[\,Q_t(a) \;+\; c\sqrt{\dfrac{\ln t}{N_t(a)}}\,\right]\,}
$$

where:
- $Q_t(a)$ = current action-value estimate
- $N_t(a)$ = number of times action $a$ has been chosen so far
- $t$ = current time-step (total plays so far)
- $c > 0$ = **confidence parameter** (statistical, *set per experiment*, not a learning hyperparameter)
- The square-root term is the **measure of uncertainty** around $a$.

### Worked example (9 trials, find action for t=10)
Rewards observed (action, reward): (A1,3) (A2,2) (A1,2) (A3,2) (A4,2) (A3,1) (A1,2) (A2,2) (A1,9)

Counts: $N(A_1)=4,\, N(A_2)=2,\, N(A_3)=2,\, N(A_4)=1$. Estimates:

$$
Q(A_1) = \tfrac{3+2+2+9}{4}=4,\quad Q(A_2)=2,\quad Q(A_3)=1.5,\quad Q(A_4)=2
$$

Uncertainty terms (with $c=2$):

$$
2\sqrt{\tfrac{\ln 9}{4}},\;\; 2\sqrt{\tfrac{\ln 9}{2}},\;\; 2\sqrt{\tfrac{\ln 9}{2}},\;\; 2\sqrt{\tfrac{\ln 9}{1}}
$$

Pick action maximising $Q + c\sqrt{\ln t / N(a)}$ → typically **A4** wins due to the largest uncertainty bonus.

### Why UCB beats ε-Greedy
- In ε-greedy, random exploration may revisit already-explored actions wastefully.
- UCB systematically channels exploration toward **least-explored, most-uncertain** actions.

---

## 10. Policy-Preference / Softmax Action Selection (Gradient Bandit) ⭐ (intro only)

Domain expert (or learned numerical preferences $H(a)$) → convert to probabilities via softmax:

$$
\boxed{\,\pi_t(a) \;=\; \dfrac{e^{H_t(a)}}{\sum_{b=1}^{k} e^{H_t(b)}}\,}
$$

- $\pi_t(a)$ = probability of choosing action $a$ at time $t$ — this **is** the policy.
- Maps "hard" preferences into a smooth probability distribution.
- Foundation for **policy-based** methods used in modern DRL (REINFORCE, A2C, PPO …).

---

## 11. Four Action-Selection Strategies — Summary Card

| # | Strategy | Picks action via | Strength | Weakness |
|---|---|---|---|---|
| 1 | **Greedy** | $\arg\max Q(a)$ | Simple, exploit fully | Locks onto sub-optimal action |
| 2 | **ε-Greedy** | $\arg\max Q(a)$ w.p. $1-\varepsilon$; random w.p. $\varepsilon$ | Mixes explore + exploit | Random exploration is "wasteful" |
| 3 | **Optimistic init** | Greedy + high $Q_1$ | Forces early broad exploration | Only short-lived; stationary only |
| 4 | **UCB** | $\arg\max [Q(a)+c\sqrt{\ln t / N(a)}]$ | Smart, uncertainty-driven exploration | Needs counts, single state only |
| (5) | **Softmax / policy** | $\pi(a)\propto e^{H(a)}$ | Stochastic, gradient-trainable | More moving parts |

---

## 12. Formula Cheat-Sheet

| # | Formula | Meaning |
|---|---|---|
| 1 | $Q_t(a)=\dfrac{\sum R}{N(a)}$ | Sample mean Q |
| 2 | $Q_{n+1}=Q_n+\tfrac{1}{n}(R_n-Q_n)$ | Stationary incremental update |
| 3 | $Q_{n+1}=Q_n+\alpha(R_n-Q_n)$ | Non-stationary (constant α) |
| 4 | $Q_{n+1}=(1-\alpha)^n Q_1+\sum \alpha(1-\alpha)^{n-i}R_i$ | Exponentially weighted form |
| 5 | $P(\text{greedy})=(1-\varepsilon)+\varepsilon/k$ | ε-Greedy greedy-probability |
| 6 | $A_t=\arg\max_a[Q_t(a)+c\sqrt{\ln t/N_t(a)}]$ | **UCB** |
| 7 | $\pi(a)=e^{H(a)}/\sum_b e^{H(b)}$ | Softmax policy |

---

*End of Session 3 summary — Session 4 moves into full MDPs.*
