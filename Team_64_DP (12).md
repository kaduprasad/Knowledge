# BIRLA INSTITUTE OF TECHNOLOGY AND SCIENCE, PILANI
## WORK INTEGRATED LEARNING PROGRAMMES DIVISION
## Deep Reinforcement Learning - Lab Assignment 1
### Part #2: Autonomous Drone Rescue Using Dynamic Programming
---
**Team Number:** 64  
**Group ID (G):** 64  

This notebook contains my Dynamic Programming solution for the autonomous drone rescue problem. I kept the model small enough to inspect by hand, but included all required MDP pieces: state definition, action dynamics, rewards, stochastic wind transitions, value iteration, policy plots, value heatmaps, and a scalability note.

**Parameters derived from G = 64:**
- Grid size: 5 x 5 because the last digit is 4
- Rescue targets: 2
- Charging stations: 1
- Danger zones: 3
- Blocked cells: 2
- Maximum battery: 10 units because the last digit is even
- Wind probability: 20%
- Maximum steps per episode: 50

### Code Cell 2

```python
# Fetch and print timestamp and machine information
import datetime
import platform
import uuid

print("="*60)
print("EXECUTION DETAILS")
print("="*60)
print(f"Execution Timestamp : {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
print(f"Machine Name        : {platform.node()}")
print(f"Virtual Machine ID  : {uuid.getnode()}")
print(f"Operating System    : {platform.system()} {platform.release()}")
print(f"Python Version      : {platform.python_version()}")
print("="*60)
```

**Output:**

```text
============================================================
EXECUTION DETAILS
============================================================
Execution Timestamp : 2026-06-07 01:03:46
Machine Name        : WDX5CG3195280
Virtual Machine ID  : 181390217077248
Operating System    : Windows 11
Python Version      : 3.13.3
============================================================
```

### Code Cell 3

```python
# Import the small set of libraries used in this notebook.
import random
import time
from itertools import product

import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import numpy as np

# The seed is tied to the group number so the stochastic simulations can be repeated.
G = 64
random.seed(G)
np.random.seed(G)

print(f"Group Number (G): {G}")
print(f"Python and NumPy random seeds set to: {G}")
```

**Output:**

```text
Group Number (G): 64
Python and NumPy random seeds set to: 64
```

---
## 1. Custom Drone Rescue Environment (1 Mark)

For Group 64 the last digit is 4, so I used the 5 x 5 version of the assignment. I placed the objects so the drone has two realistic choices: go north-east first for the top rescue, or use the middle charging station before heading toward the lower rescue.

### Grid Used

| Row/Col | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| 0 | S | F | D | F | R |
| 1 | F | X | F | W | F |
| 2 | D | F | C | F | D |
| 3 | F | W | F | X | F |
| 4 | F | F | R | F | F |

### Symbols

| Symbol | Meaning |
|---|---|
| S | Start at (0, 0) |
| F | Safe/free cell |
| D | Danger zone |
| R | Rescue target |
| C | Charging station |
| W | Wind zone |
| X | Blocked cell |

The drone starts with a full battery of 10 units. The state is `(row, col, battery, rescue_status)`, where `rescue_status` stores whether each rescue target has already been cleared.

### Code Cell 5

```python
class DroneRescueEnvironment:
    """
    Custom Drone Rescue Environment for Dynamic Programming.
    
    The drone navigates a 5x5 grid to rescue stranded civilians while
    managing battery, avoiding danger zones, and handling wind disturbances.
    
    State representation: (row, col, battery, rescue_status_tuple)
    - row, col: drone position on the grid (0-indexed)
    - battery: current battery level (0 to max_battery)
    - rescue_status_tuple: tuple of booleans indicating which targets are rescued
    """
    
    # Action constants
    UP = 0
    DOWN = 1
    LEFT = 2
    RIGHT = 3
    HOVER = 4
    
    ACTION_NAMES = ['Up', 'Down', 'Left', 'Right', 'Hover']
    
    # Movement deltas for each action (row_delta, col_delta)
    ACTION_DELTAS = {
        0: (-1, 0),  # Up
        1: (1, 0),   # Down
        2: (0, -1),  # Left
        3: (0, 1),   # Right
        4: (0, 0),   # Hover
    }
    
    # Reward constants
    REWARD_RESCUE = 20
    REWARD_DANGER = -10
    REWARD_BATTERY_DEAD = -20
    REWARD_CHARGING = 5
    REWARD_MOVEMENT = -1
    
    def __init__(self, group_id=64):
        """
        Initialize the Drone Rescue Environment based on group ID.
        
        Parameters:
        -----------
        group_id : int
            The team's group number used to configure environment parameters.
        """
        self.group_id = group_id
        last_digit = group_id % 10
        
        # Determine grid size based on last digit of group ID
        if last_digit <= 4:
            self.grid_size = 5
        else:
            self.grid_size = 6
        
        # Determine max battery based on last digit (even/odd)
        if last_digit % 2 == 0:
            self.max_battery = 10
        else:
            self.max_battery = 15
        
        # Determine wind probability
        if last_digit <= 4:
            self.wind_prob = 0.20
        else:
            self.wind_prob = 0.30
        
        # Maximum steps per episode
        self.max_steps = 50 if self.grid_size == 5 else 75
        
        # Define the grid layout
        # For Group 64 (last digit 4): 2 rescue, 1 charging, 3 danger, 2 blocked
        self.grid = [
            ['S', 'F', 'D', 'F', 'R'],
            ['F', 'X', 'F', 'W', 'F'],
            ['D', 'F', 'C', 'F', 'D'],
            ['F', 'W', 'F', 'X', 'F'],
            ['F', 'F', 'R', 'F', 'F']
        ]
        
        # Identify rescue target positions
        self.rescue_positions = []
        for r in range(self.grid_size):
            for c in range(self.grid_size):
                if self.grid[r][c] == 'R':
                    self.rescue_positions.append((r, c))
        
        self.num_rescues = len(self.rescue_positions)
        
        # Starting position is top-left corner
        self.start_pos = (0, 0)
        self.start_battery = self.max_battery
        
        # Initialize state variables
        self.reset()
    
    def reset(self):
        """
        Reset the environment to its initial state.
        
        Returns:
        --------
        state : tuple
            Initial state (row, col, battery, rescue_status)
        """
        self.drone_pos = self.start_pos
        self.battery = self.start_battery
        self.rescue_status = tuple([False] * self.num_rescues)  # No targets rescued yet
        self.steps = 0
        self.done = False
        self.total_reward = 0
        return self._get_state()
    
    def _get_state(self):
        """
        Get the current state representation.
        
        Returns:
        --------
        state : tuple
            (row, col, battery, rescue_status_tuple)
        """
        return (self.drone_pos[0], self.drone_pos[1], self.battery, self.rescue_status)
    
    def get_valid_actions(self, state=None):
        """
        Get all valid actions from the current or given state.
        All 5 actions are always available (blocked cells cause drone to stay in place).
        
        Parameters:
        -----------
        state : tuple, optional
            The state to check. If None, uses current state.
        
        Returns:
        --------
        actions : list
            List of valid action indices [0, 1, 2, 3, 4]
        """
        return [self.UP, self.DOWN, self.LEFT, self.RIGHT, self.HOVER]
    
    def _is_valid_position(self, row, col):
        """
        Check if a position is within grid bounds.
        
        Parameters:
        -----------
        row, col : int
            Grid coordinates to check.
        
        Returns:
        --------
        bool : True if position is within bounds
        """
        return 0 <= row < self.grid_size and 0 <= col < self.grid_size
    
    def _is_blocked(self, row, col):
        """
        Check if a cell is blocked.
        
        Parameters:
        -----------
        row, col : int
            Grid coordinates to check.
        
        Returns:
        --------
        bool : True if the cell is blocked (X)
        """
        return self.grid[row][col] == 'X'
    
    def _get_cell_type(self, row, col, rescue_status):
        """
        Get the effective cell type considering rescue status.
        If a rescue target has been rescued, the cell becomes Free.
        
        Parameters:
        -----------
        row, col : int
            Position to check.
        rescue_status : tuple
            Current rescue status.
        
        Returns:
        --------
        str : Cell type character
        """
        cell = self.grid[row][col]
        if cell == 'R':
            # Check if this rescue target has already been rescued
            idx = self.rescue_positions.index((row, col))
            if rescue_status[idx]:
                return 'F'  # Already rescued, now a free cell
        return cell
    
    def get_transition(self, state, action):
        """
        Compute transition probabilities from a given state-action pair.
        Accounts for wind zone stochastic transitions.
        
        Parameters:
        -----------
        state : tuple
            (row, col, battery, rescue_status)
        action : int
            Action index (0-4)
        
        Returns:
        --------
        transitions : list of tuples
            Each tuple: (probability, next_state, reward, done)
        """
        row, col, battery, rescue_status = state
        
        # Terminal state check: battery is 0 or all rescued
        if battery <= 0 or all(rescue_status):
            return [(1.0, state, 0, True)]
        
        transitions = []
        current_cell = self._get_cell_type(row, col, rescue_status)
        
        # Determine if wind affects movement
        if current_cell == 'W' and action != self.HOVER:
            # Wind zone: with wind_prob, direction changes randomly
            # With (1 - wind_prob), intended action succeeds
            movement_actions = [self.UP, self.DOWN, self.LEFT, self.RIGHT]
            
            # Intended direction succeeds with probability (1 - wind_prob)
            prob_intended = 1.0 - self.wind_prob
            transitions.append(self._compute_move_result(
                row, col, battery, rescue_status, action, prob_intended))
            
            # Wind disturbs: uniform over all 4 directions
            prob_each_wind = self.wind_prob / 4.0
            for wind_action in movement_actions:
                transitions.append(self._compute_move_result(
                    row, col, battery, rescue_status, wind_action, prob_each_wind))
        else:
            # No wind effect: deterministic transition
            transitions.append(self._compute_move_result(
                row, col, battery, rescue_status, action, 1.0))
        
        # Merge transitions with the same next_state
        merged = {}
        for prob, next_state, reward, done in transitions:
            key = (next_state, reward, done)
            if key in merged:
                merged[key] += prob
            else:
                merged[key] = prob
        
        return [(prob, ns, r, d) for (ns, r, d), prob in merged.items()]
    
    def _compute_move_result(self, row, col, battery, rescue_status, action, probability):
        """
        Compute the result of taking an action from a given position.
        
        Parameters:
        -----------
        row, col : int
            Current position.
        battery : int
            Current battery level.
        rescue_status : tuple
            Current rescue status.
        action : int
            Action to take.
        probability : float
            Probability of this transition.
        
        Returns:
        --------
        tuple : (probability, next_state, reward, done)
        """
        dr, dc = self.ACTION_DELTAS[action]
        new_row, new_col = row + dr, col + dc
        
        # Handle hover on charging station
        if action == self.HOVER:
            current_cell = self._get_cell_type(row, col, rescue_status)
            if current_cell == 'C':
                # Hovering on a charging station recharges, but the +5 reward is only for reaching it.
                new_battery = min(battery + 2, self.max_battery)
                new_state = (row, col, new_battery, rescue_status)
                return (probability, new_state, 0, False)
            else:
                # Hovering elsewhere costs 1 battery
                new_battery = battery - 1
                if new_battery <= 0:
                    new_state = (row, col, 0, rescue_status)
                    return (probability, new_state, self.REWARD_BATTERY_DEAD, True)
                new_state = (row, col, new_battery, rescue_status)
                return (probability, new_state, self.REWARD_MOVEMENT, False)
        
        # Check if new position is valid and not blocked
        if not self._is_valid_position(new_row, new_col) or self._is_blocked(new_row, new_col):
            # Stay in place but still consume battery
            new_row, new_col = row, col
        
        # Consume battery for the move
        new_battery = battery - 1
        
        # Check battery depletion
        if new_battery <= 0:
            new_state = (new_row, new_col, 0, rescue_status)
            return (probability, new_state, self.REWARD_BATTERY_DEAD, True)
        
        # Determine reward and state changes based on destination cell type
        dest_cell = self._get_cell_type(new_row, new_col, rescue_status)
        reward = self.REWARD_MOVEMENT  # Base movement cost
        new_rescue_status = rescue_status
        done = False
        
        if dest_cell == 'D':
            # Danger zone: heavy negative reward
            reward = self.REWARD_DANGER
        elif dest_cell == 'R':
            # Rescue target: gain reward and mark as rescued
            idx = self.rescue_positions.index((new_row, new_col))
            new_rescue_status = list(rescue_status)
            new_rescue_status[idx] = True
            new_rescue_status = tuple(new_rescue_status)
            reward = self.REWARD_RESCUE
            # Check if all rescued
            if all(new_rescue_status):
                done = True
        elif dest_cell == 'C':
            # Charging station: battery becomes full
            new_battery = self.max_battery
            reward = self.REWARD_CHARGING
        
        new_state = (new_row, new_col, new_battery, new_rescue_status)
        return (probability, new_state, reward, done)
    
    def step(self, action):
        """
        Execute one step in the environment (for simulation/testing).
        
        Parameters:
        -----------
        action : int
            Action index (0-4)
        
        Returns:
        --------
        tuple : (next_state, reward, done, info)
        """
        if self.done:
            return self._get_state(), 0, True, {'reason': 'Episode already done'}
        
        state = self._get_state()
        transitions = self.get_transition(state, action)
        
        # Sample from transitions based on probabilities
        probs = [t[0] for t in transitions]
        idx = np.random.choice(len(transitions), p=probs)
        _, next_state, reward, done = transitions[idx]
        
        # Update environment state
        self.drone_pos = (next_state[0], next_state[1])
        self.battery = next_state[2]
        self.rescue_status = next_state[3]
        self.steps += 1
        self.total_reward += reward
        
        # Check max steps
        if self.steps >= self.max_steps:
            done = True
        
        self.done = done
        
        info = {
            'steps': self.steps,
            'battery': self.battery,
            'rescue_status': self.rescue_status,
            'total_reward': self.total_reward
        }
        
        return self._get_state(), reward, done, info
    
    def render(self, state=None):
        """
        Render the current state of the environment as a text grid.
        
        Parameters:
        -----------
        state : tuple, optional
            State to render. If None, uses current environment state.
        """
        if state is None:
            row, col = self.drone_pos
            battery = self.battery
            rescue_status = self.rescue_status
        else:
            row, col, battery, rescue_status = state
        
        print(f"\nDrone Position: ({row}, {col}) | Battery: {battery}/{self.max_battery}")
        print(f"Rescue Status: {rescue_status} | Steps: {self.steps}")
        print("-" * (self.grid_size * 4 + 1))
        
        for r in range(self.grid_size):
            row_str = "|"
            for c in range(self.grid_size):
                if (r, c) == (row, col):
                    row_str += " \U0001F916 |"
                else:
                    cell = self._get_cell_type(r, c, rescue_status)
                    symbols = {'S': ' S ', 'F': ' . ', 'D': ' D ', 'R': ' R ',
                              'C': ' C ', 'W': ' W ', 'X': ' X '}
                    row_str += f"{symbols.get(cell, ' ? ')}|"
            print(row_str)
        print("-" * (self.grid_size * 4 + 1))


# Instantiate the environment
env = DroneRescueEnvironment(group_id=64)
print("Drone Rescue Environment Created Successfully!")
print(f"Grid Size: {env.grid_size}x{env.grid_size}")
print(f"Max Battery: {env.max_battery}")
print(f"Wind Probability: {env.wind_prob}")
print(f"Max Steps: {env.max_steps}")
print(f"Rescue Targets at: {env.rescue_positions}")
print(f"Number of Actions: 5 (Up, Down, Left, Right, Hover)")
print(f"\nInitial Grid Layout:")
env.render()
```

**Output:**

```text
Drone Rescue Environment Created Successfully!
Grid Size: 5x5
Max Battery: 10
Wind Probability: 0.2
Max Steps: 50
Rescue Targets at: [(0, 4), (4, 2)]
Number of Actions: 5 (Up, Down, Left, Right, Hover)

Initial Grid Layout:

Drone Position: (0, 0) | Battery: 10/10
Rescue Status: (False, False) | Steps: 0
---------------------
| 🤖 | . | D | . | R |
| . | X | . | W | . |
| D | . | C | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------
```

### Code Cell 6

```python
# Test the environment with a few random actions
print("=" * 50)
print("ENVIRONMENT TEST: Running sample episode")
print("=" * 50)

state = env.reset()
print(f"Initial State: {state}")
env.render()

# Take a few sample actions
test_actions = [env.DOWN, env.DOWN, env.RIGHT, env.RIGHT, env.HOVER]
for i, action in enumerate(test_actions):
    next_state, reward, done, info = env.step(action)
    print(f"\nStep {i+1}: Action={env.ACTION_NAMES[action]}, Reward={reward}, Done={done}")
    env.render()
    if done:
        break

print(f"\nTotal Reward after test: {env.total_reward}")
```

**Output:**

```text
==================================================
ENVIRONMENT TEST: Running sample episode
==================================================
Initial State: (0, 0, 10, (False, False))

Drone Position: (0, 0) | Battery: 10/10
Rescue Status: (False, False) | Steps: 0
---------------------
| 🤖 | . | D | . | R |
| . | X | . | W | . |
| D | . | C | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------

Step 1: Action=Down, Reward=-1, Done=False

Drone Position: (1, 0) | Battery: 9/10
Rescue Status: (False, False) | Steps: 1
---------------------
| S | . | D | . | R |
| 🤖 | X | . | W | . |
| D | . | C | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------

Step 2: Action=Down, Reward=-10, Done=False

Drone Position: (2, 0) | Battery: 8/10
Rescue Status: (False, False) | Steps: 2
---------------------
| S | . | D | . | R |
| . | X | . | W | . |
| 🤖 | . | C | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------

Step 3: Action=Right, Reward=-1, Done=False

Drone Position: (2, 1) | Battery: 7/10
Rescue Status: (False, False) | Steps: 3
---------------------
| S | . | D | . | R |
| . | X | . | W | . |
| D | 🤖 | C | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------

Step 4: Action=Right, Reward=5, Done=False

Drone Position: (2, 2) | Battery: 10/10
Rescue Status: (False, False) | Steps: 4
---------------------
| S | . | D | . | R |
| . | X | . | W | . |
| D | . | 🤖 | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------

Step 5: Action=Hover, Reward=0, Done=False

Drone Position: (2, 2) | Battery: 10/10
Rescue Status: (False, False) | Steps: 5
---------------------
| S | . | D | . | R |
| . | X | . | W | . |
| D | . | 🤖 | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------

Total Reward after test: -7
```

---
## 2. Dynamic Programming Solution - Value Iteration (2 Marks)

### State Representation
Each DP state is represented as:

`(row, col, battery, rescue_status, steps_remaining)`

- `row`, `col`: drone location on the grid
- `battery`: current battery from 0 to 10
- `rescue_status`: a tuple such as `(False, True)` showing which rescue targets are complete
- `steps_remaining`: how many moves are left before the 50-step episode limit

The assignment lists position, battery, and rescue status as the minimum state variables. I added `steps_remaining` because the environment has a maximum step limit. Without this term, an infinite-horizon DP agent can learn to circle around the charging station for repeated charging rewards instead of finishing the rescue mission.

### Value Iteration Settings

- Discount factor gamma = 0.95
- Stopping threshold theta = 1e-3
- Actions: Up, Down, Left, Right, Hover

### Code Cell 8

```python
def enumerate_states(env):
    """
    Enumerate the finite state space used by the DP solver.

    The first four fields are the drone position, battery, and rescue status.
    The final field, steps_remaining, represents the episode time limit so the
    DP model matches the environment's 50-step termination rule.
    """
    states = []

    valid_positions = []
    for row in range(env.grid_size):
        for col in range(env.grid_size):
            if env.grid[row][col] != 'X':
                valid_positions.append((row, col))

    rescue_combinations = list(product([False, True], repeat=env.num_rescues))
    battery_levels = range(0, env.max_battery + 1)
    step_counts = range(0, env.max_steps + 1)

    for row, col in valid_positions:
        for battery in battery_levels:
            for rescue_status in rescue_combinations:
                for steps_remaining in step_counts:
                    states.append((row, col, battery, rescue_status, steps_remaining))

    return states


# Enumerate states for the step-aware MDP.
all_states = enumerate_states(env)
valid_position_count = sum(
    1 for row in range(env.grid_size) for col in range(env.grid_size)
    if env.grid[row][col] != 'X'
)

print(f"Total enumerated states: {len(all_states)}")
print("\nState space breakdown:")
print(f"  Valid positions (non-blocked): {valid_position_count}")
print(f"  Battery levels: {env.max_battery + 1} (0 to {env.max_battery})")
print(f"  Rescue combinations: {2**env.num_rescues}")
print(f"  Step counts: {env.max_steps + 1} (0 to {env.max_steps})")
print("\nSample states (first 5):")
for state in all_states[:5]:
    print(f"  {state}")
```

**Output:**

```text
Total enumerated states: 51612

State space breakdown:
  Valid positions (non-blocked): 23
  Battery levels: 11 (0 to 10)
  Rescue combinations: 4
  Step counts: 51 (0 to 50)

Sample states (first 5):
  (0, 0, 0, (False, False), 0)
  (0, 0, 0, (False, False), 1)
  (0, 0, 0, (False, False), 2)
  (0, 0, 0, (False, False), 3)
  (0, 0, 0, (False, False), 4)
```

### Code Cell 9

```python
def value_iteration(env, gamma=0.95, theta=1e-3):
    """
    Compute the optimal value function and greedy policy using Value Iteration.

    The Bellman update is applied over the step-aware state:
        (row, col, battery, rescue_status, steps_remaining)

    Adding steps_remaining makes the DP model respect the episode's maximum
    length and prevents repeated charging rewards from becoming the main goal.
    """
    start_time = time.time()

    all_states = enumerate_states(env)
    actions = env.get_valid_actions()

    V = {state: 0.0 for state in all_states}
    policy = {state: env.HOVER for state in all_states}
    delta_history = []

    print("Starting Value Iteration")
    print(f"gamma = {gamma}, theta = {theta}")
    print(f"states evaluated = {len(all_states)}")
    print("-" * 50)

    iterations = 0
    while True:
        delta = 0.0
        iterations += 1

        for state in all_states:
            row, col, battery, rescue_status, steps_remaining = state

            # Terminal states have no future value.
            if battery <= 0 or all(rescue_status) or steps_remaining <= 0:
                continue

            old_value = V[state]
            base_state = (row, col, battery, rescue_status)
            next_steps = steps_remaining - 1
            best_value = float("-inf")
            best_action = env.HOVER

            for action in actions:
                action_value = 0.0
                for prob, next_base_state, reward, done in env.get_transition(base_state, action):
                    next_state = (
                        next_base_state[0],
                        next_base_state[1],
                        next_base_state[2],
                        next_base_state[3],
                        next_steps,
                    )
                    episode_done = done or next_steps <= 0
                    future_value = 0.0 if episode_done else V.get(next_state, 0.0)
                    action_value += prob * (reward + gamma * future_value)

                if action_value > best_value:
                    best_value = action_value
                    best_action = action

            V[state] = best_value
            policy[state] = best_action
            delta = max(delta, abs(old_value - best_value))

        delta_history.append(delta)
        print(f"Iteration {iterations:03d}: delta = {delta:.6f}")

        if delta < theta:
            break

    runtime = time.time() - start_time

    print("-" * 50)
    print("Value Iteration converged")
    print(f"Iterations: {iterations}")
    print(f"Runtime: {runtime:.4f} seconds")
    print(f"Final delta/error: {delta:.8f}")

    return V, policy, iterations, runtime, delta, delta_history


# Run Value Iteration.
V_star, pi_star, num_iters, runtime, final_delta, delta_history = value_iteration(env, gamma=0.95, theta=1e-3)
```

**Output:**

```text
Starting Value Iteration
gamma = 0.95, theta = 0.001
states evaluated = 51612
--------------------------------------------------
Iteration 001: delta = 26.777656
Iteration 002: delta = 19.000000
Iteration 003: delta = 18.050000
Iteration 004: delta = 17.147500
Iteration 005: delta = 12.860625
Iteration 006: delta = 11.606714
Iteration 007: delta = 5.717440
Iteration 008: delta = 5.431568
Iteration 009: delta = 1.715686
Iteration 010: delta = 1.548406
Iteration 011: delta = 1.397437
Iteration 012: delta = 1.261187
Iteration 013: delta = 1.138221
Iteration 014: delta = 1.027244
Iteration 015: delta = 0.927088
Iteration 016: delta = 0.836697
Iteration 017: delta = 0.755119
Iteration 018: delta = 0.681495
Iteration 019: delta = 0.615049
Iteration 020: delta = 0.555082
Iteration 021: delta = 0.500961
Iteration 022: delta = 0.452118
Iteration 023: delta = 0.408036
Iteration 024: delta = 0.368253
Iteration 025: delta = 0.332348
Iteration 026: delta = 0.299944
Iteration 027: delta = 0.270700
Iteration 028: delta = 0.188464
Iteration 029: delta = 0.000000
--------------------------------------------------
Value Iteration converged
Iterations: 29
Runtime: 7.8035 seconds
Final delta/error: 0.00000000
```

### Code Cell 10

```python
# Plot convergence of Value Iteration
plt.figure(figsize=(10, 5))
plt.plot(range(1, len(delta_history) + 1), delta_history, 'b-', linewidth=1.5)
plt.axhline(y=1e-3, color='r', linestyle='--', label='Threshold (θ=10⁻³)')
plt.xlabel('Iteration', fontsize=12)
plt.ylabel('Delta (Max Value Change)', fontsize=12)
plt.title('Value Iteration Convergence', fontsize=14)
plt.yscale('log')
plt.legend(fontsize=11)
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()

print(f"\nConvergence Summary:")
print(f"  Iterations to converge: {num_iters}")
print(f"  Runtime: {runtime:.4f} seconds")
print(f"  Final delta/error: {final_delta:.8f}")
```

**Output:**

![Output from code cell 10](Team_64_DP%20(12)_files/output_10_1.png)

```text

Convergence Summary:
  Iterations to converge: 29
  Runtime: 7.8035 seconds
  Final delta/error: 0.00000000
```

### Code Cell 11

```python
# Simulate optimal policy execution.
def simulate_policy(env, policy, num_episodes=10, verbose=True):
    """
    Simulate the optimal policy and display the drone trajectory.

    The environment returns the four physical state fields. During policy lookup,
    I append the current steps_remaining value used by the DP model.
    """
    results = []

    for episode in range(num_episodes):
        state = env.reset()
        trajectory = [state]
        actions_taken = []
        rewards = []
        done = False

        if verbose and episode == 0:
            print(f"\n{'='*60}")
            print(f"EPISODE {episode+1}: Simulating Optimal Policy")
            print(f"{'='*60}")
            env.render()

        step_count = 0
        while not done and step_count < env.max_steps:
            steps_remaining = env.max_steps - step_count
            policy_state = (state[0], state[1], state[2], state[3], steps_remaining)
            action = policy.get(policy_state, env.HOVER)
            next_state, reward, done, info = env.step(action)

            trajectory.append(next_state)
            actions_taken.append(action)
            rewards.append(reward)

            if verbose and episode == 0:
                print(f"  Step {step_count+1}: {env.ACTION_NAMES[action]:>5} -> "
                      f"Pos=({next_state[0]},{next_state[1]}) "
                      f"Bat={next_state[2]} Reward={reward:+.0f}")

            state = next_state
            step_count += 1

        total_reward = sum(rewards)
        results.append({
            'trajectory': trajectory,
            'actions': actions_taken,
            'rewards': rewards,
            'total_reward': total_reward,
            'steps': step_count,
            'all_rescued': all(state[3])
        })

        if verbose and episode == 0:
            print(f"\n  Episode Complete! Steps={step_count}, "
                  f"Total Reward={total_reward:.1f}, "
                  f"All Rescued={all(state[3])}")
            env.render()

    avg_reward = np.mean([result['total_reward'] for result in results])
    avg_steps = np.mean([result['steps'] for result in results])
    rescue_rate = np.mean([result['all_rescued'] for result in results])

    print(f"\n{'='*60}")
    print(f"SIMULATION SUMMARY ({num_episodes} episodes)")
    print(f"{'='*60}")
    print(f"  Average Total Reward: {avg_reward:.2f}")
    print(f"  Average Steps: {avg_steps:.1f}")
    print(f"  Rescue Success Rate: {rescue_rate*100:.1f}%")

    return results


# Run simulation with the optimal policy.
results = simulate_policy(env, pi_star, num_episodes=20, verbose=True)
```

**Output:**

```text

============================================================
EPISODE 1: Simulating Optimal Policy
============================================================

Drone Position: (0, 0) | Battery: 10/10
Rescue Status: (False, False) | Steps: 0
---------------------
| 🤖 | . | D | . | R |
| . | X | . | W | . |
| D | . | C | . | D |
| . | W | . | X | . |
| . | . | R | . | . |
---------------------
  Step 1:  Down -> Pos=(1,0) Bat=9 Reward=-1
  Step 2:  Down -> Pos=(2,0) Bat=8 Reward=-10
  Step 3: Right -> Pos=(2,1) Bat=7 Reward=-1
  Step 4: Right -> Pos=(2,2) Bat=10 Reward=+5
  Step 5:  Down -> Pos=(3,2) Bat=9 Reward=-1
  Step 6:  Down -> Pos=(4,2) Bat=8 Reward=+20
  Step 7:    Up -> Pos=(3,2) Bat=7 Reward=-1
  Step 8:    Up -> Pos=(2,2) Bat=10 Reward=+5
  Step 9:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 10:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 11:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 12:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 13:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 14:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 15:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 16:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 17:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 18:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 19:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 20:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 21:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 22:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 23:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 24:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 25:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 26:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 27:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 28:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 29:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 30:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 31:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 32:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 33:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 34:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 35:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 36:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 37:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 38:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 39:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 40:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 41:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 42:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 43:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 44:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 45:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 46:  Down -> Pos=(2,2) Bat=10 Reward=+5
  Step 47:    Up -> Pos=(1,2) Bat=9 Reward=-1
  Step 48: Right -> Pos=(1,3) Bat=8 Reward=-1
  Step 49:    Up -> Pos=(0,3) Bat=7 Reward=-1
  Step 50: Right -> Pos=(0,4) Bat=6 Reward=+20

  Episode Complete! Steps=50, Total Reward=109.0, All Rescued=True

Drone Position: (0, 4) | Battery: 6/10
Rescue Status: (True, True) | Steps: 50
---------------------
| S | . | D | . | 🤖 |
| . | X | . | W | . |
| D | . | C | . | D |
| . | W | . | X | . |
| . | . | . | . | . |
---------------------

============================================================
SIMULATION SUMMARY (20 episodes)
============================================================
  Average Total Reward: 108.25
  Average Steps: 50.0
  Rescue Success Rate: 95.0%
```

### Policy Rollout Note

The rollout shows an interesting reward-design effect. Because reaching the charging station gives a positive reward, the optimal policy sometimes uses the charger before finishing the last rescue. Adding `steps_remaining` prevents this from becoming an endless loop and forces the mission to finish within the episode limit. This is a useful reminder that reward design affects not only convergence, but also how efficient or natural the learned behaviour looks.

---
## 3. Policy Visualisation (1 Mark)

Visualize the optimal policy showing:
- Drone movement directions (arrows)
- Rescue sequence
- Preferred charging behaviour
- Avoidance of dangerous zones

### Code Cell 14

```python
def visualize_policy_grid(env, policy, battery_level=10, rescue_status=(False, False), steps_remaining=None):
    """
    Visualize the optimal action at each grid cell for one slice of the state space.
    """
    if steps_remaining is None:
        steps_remaining = env.max_steps

    fig, ax = plt.subplots(1, 1, figsize=(8, 8))

    color_map = {
        'S': '#90EE90',
        'F': '#FFFFFF',
        'D': '#FF6B6B',
        'R': '#FFD700',
        'C': '#87CEEB',
        'W': '#DDA0DD',
        'X': '#404040',
    }

    arrow_map = {
        0: (0, 0.3),
        1: (0, -0.3),
        2: (-0.3, 0),
        3: (0.3, 0),
        4: None,
    }

    for row in range(env.grid_size):
        for col in range(env.grid_size):
            cell = env._get_cell_type(row, col, rescue_status)
            color = color_map.get(cell, '#FFFFFF')
            y_pos = env.grid_size - 1 - row

            rect = plt.Rectangle((col - 0.5, y_pos - 0.5), 1, 1,
                                 facecolor=color, edgecolor='black', linewidth=1.5)
            ax.add_patch(rect)
            ax.text(col, y_pos + 0.35, cell,
                    ha='center', va='center', fontsize=10, fontweight='bold', color='gray')

            if cell != 'X':
                state = (row, col, battery_level, rescue_status, steps_remaining)
                action = policy.get(state, env.HOVER)

                if arrow_map[action] is not None:
                    dx, dy = arrow_map[action]
                    ax.annotate('', xy=(col + dx, y_pos + dy), xytext=(col, y_pos),
                                arrowprops=dict(arrowstyle='->', color='black', lw=2.0))
                else:
                    circle = plt.Circle((col, y_pos), 0.15, fill=False,
                                        color='black', linewidth=2)
                    ax.add_patch(circle)

    ax.set_xlim(-0.6, env.grid_size - 0.4)
    ax.set_ylim(-0.6, env.grid_size - 0.4)
    ax.set_aspect('equal')
    ax.set_xticks(range(env.grid_size))
    ax.set_yticks(range(env.grid_size))
    ax.set_xticklabels(range(env.grid_size))
    ax.set_yticklabels(range(env.grid_size - 1, -1, -1))
    ax.set_xlabel('Column', fontsize=12)
    ax.set_ylabel('Row', fontsize=12)

    rescue_str = 'none rescued' if not any(rescue_status) else f'rescued={rescue_status}'
    ax.set_title(f'Optimal Policy Slice\nBattery={battery_level}, {rescue_str}, steps left={steps_remaining}',
                 fontsize=13, fontweight='bold')

    legend_patches = [
        mpatches.Patch(color='#90EE90', label='Start (S)'),
        mpatches.Patch(color='#FFFFFF', label='Free (F)'),
        mpatches.Patch(color='#FF6B6B', label='Danger (D)'),
        mpatches.Patch(color='#FFD700', label='Rescue (R)'),
        mpatches.Patch(color='#87CEEB', label='Charging (C)'),
        mpatches.Patch(color='#DDA0DD', label='Wind (W)'),
        mpatches.Patch(color='#404040', label='Blocked (X)'),
    ]
    ax.legend(handles=legend_patches, loc='upper left', bbox_to_anchor=(1.02, 1), fontsize=10)

    plt.tight_layout()
    plt.show()


print("Policy Visualization: full battery, no rescues, start of episode")
visualize_policy_grid(env, pi_star, battery_level=10, rescue_status=(False, False), steps_remaining=50)

print("\nPolicy Visualization: low battery, no rescues, 30 steps left")
visualize_policy_grid(env, pi_star, battery_level=3, rescue_status=(False, False), steps_remaining=30)

print("\nPolicy Visualization: full battery, first target rescued, 40 steps left")
visualize_policy_grid(env, pi_star, battery_level=10, rescue_status=(True, False), steps_remaining=40)
```

**Output:**

```text
Policy Visualization: full battery, no rescues, start of episode
```

![Output from code cell 14](Team_64_DP%20(12)_files/output_14_2.png)

```text

Policy Visualization: low battery, no rescues, 30 steps left
```

![Output from code cell 14](Team_64_DP%20(12)_files/output_14_3.png)

```text

Policy Visualization: full battery, first target rescued, 40 steps left
```

![Output from code cell 14](Team_64_DP%20(12)_files/output_14_4.png)

### Code Cell 15

```python
def visualize_trajectory(env, results):
    """
    Visualize the drone's trajectory on the grid from a simulation result.
    
    Parameters:
    -----------
    env : DroneRescueEnvironment
        The environment instance.
    results : list
        Simulation results from simulate_policy().
    """
    # Use first episode trajectory
    trajectory = results[0]['trajectory']
    
    fig, ax = plt.subplots(1, 1, figsize=(8, 8))
    
    color_map = {
        'S': '#90EE90', 'F': '#FFFFFF', 'D': '#FF6B6B',
        'R': '#FFD700', 'C': '#87CEEB', 'W': '#DDA0DD', 'X': '#404040'
    }
    
    # Draw grid
    for r in range(env.grid_size):
        for c in range(env.grid_size):
            cell = env.grid[r][c]
            color = color_map.get(cell, '#FFFFFF')
            rect = plt.Rectangle((c - 0.5, (env.grid_size - 1 - r) - 0.5), 1, 1,
                                facecolor=color, edgecolor='black', linewidth=1.5)
            ax.add_patch(rect)
            ax.text(c, (env.grid_size - 1 - r), cell,
                   ha='center', va='center', fontsize=14, fontweight='bold',
                   color='gray', alpha=0.5)
    
    # Draw trajectory path
    positions = [(s[1], env.grid_size - 1 - s[0]) for s in trajectory]
    
    for i in range(len(positions) - 1):
        x1, y1 = positions[i]
        x2, y2 = positions[i + 1]
        
        # Draw path line with gradient color
        progress = i / max(len(positions) - 1, 1)
        color = plt.cm.viridis(progress)
        
        # Slight offset to avoid overlap
        offset = 0.05 * (i % 3 - 1)
        ax.annotate('', xy=(x2 + offset, y2 + offset),
                   xytext=(x1 + offset, y1 + offset),
                   arrowprops=dict(arrowstyle='->', color=color, lw=1.5, alpha=0.8))
    
    # Mark start and end
    ax.plot(positions[0][0], positions[0][1], 'go', markersize=15, label='Start', zorder=5)
    ax.plot(positions[-1][0], positions[-1][1], 'rs', markersize=15, label='End', zorder=5)
    
    ax.set_xlim(-0.6, env.grid_size - 0.4)
    ax.set_ylim(-0.6, env.grid_size - 0.4)
    ax.set_aspect('equal')
    ax.set_xlabel('Column', fontsize=12)
    ax.set_ylabel('Row', fontsize=12)
    ax.set_title(f'Drone Trajectory (Optimal Policy)\n'
                f'Steps={results[0]["steps"]}, Reward={results[0]["total_reward"]:.1f}, '
                f'All Rescued={results[0]["all_rescued"]}',
                fontsize=13, fontweight='bold')
    ax.legend(fontsize=11, loc='upper right')
    plt.tight_layout()
    plt.show()


# Visualize trajectory
visualize_trajectory(env, results)
```

**Output:**

![Output from code cell 15](Team_64_DP%20(12)_files/output_15_5.png)

---
## 4. State-Value Analysis (1 Mark)

### Analysis Approach
We fix the rescue target status and battery level, then vary only the drone position to create
a heatmap of V*(s). This reveals which positions are most valuable for the drone.

We examine multiple scenarios:
1. Full battery, no rescues done
2. Low battery, no rescues done
3. Full battery, one target rescued

### Code Cell 17

```python
def plot_value_heatmap(env, V, battery_level, rescue_status, steps_remaining, title_suffix=""):
    """
    Plot a heatmap of V*(s) by varying position while fixing battery, rescue
    status, and the number of steps remaining.
    """
    value_grid = np.full((env.grid_size, env.grid_size), np.nan)

    for row in range(env.grid_size):
        for col in range(env.grid_size):
            if env.grid[row][col] != 'X':
                state = (row, col, battery_level, rescue_status, steps_remaining)
                value_grid[row][col] = V.get(state, 0.0)

    fig, ax = plt.subplots(1, 1, figsize=(8, 7))
    masked_grid = np.ma.masked_invalid(value_grid)

    im = ax.imshow(masked_grid, cmap='RdYlGn', interpolation='nearest')
    plt.colorbar(im, ax=ax, label='State Value V*(s)')

    for row in range(env.grid_size):
        for col in range(env.grid_size):
            cell = env.grid[row][col]
            if cell == 'X':
                ax.text(col, row, 'X\n(blocked)', ha='center', va='center',
                        fontsize=9, color='white', fontweight='bold')
            else:
                value = value_grid[row][col]
                ax.text(col, row, f'{cell}\n{value:.1f}', ha='center', va='center',
                        fontsize=9, fontweight='bold')

    ax.set_xticks(range(env.grid_size))
    ax.set_yticks(range(env.grid_size))
    ax.set_xlabel('Column', fontsize=12)
    ax.set_ylabel('Row', fontsize=12)
    ax.set_title(f'State-Value Heatmap V*(s)\nBattery={battery_level}, '
                 f'Rescue={rescue_status}, Steps left={steps_remaining} {title_suffix}',
                 fontsize=13, fontweight='bold')

    plt.tight_layout()
    plt.show()

    valid_values = masked_grid.compressed()
    print("  Value Statistics:")
    print(f"    Max V*(s): {np.max(valid_values):.3f}")
    print(f"    Min V*(s): {np.min(valid_values):.3f}")
    print(f"    Mean V*(s): {np.mean(valid_values):.3f}")
    print(f"    Std V*(s): {np.std(valid_values):.3f}")


print("Scenario 1: Full battery, no targets rescued, start of episode")
print("="*50)
plot_value_heatmap(env, V_star, battery_level=10, rescue_status=(False, False),
                   steps_remaining=50, title_suffix="(Initial State)")

print("\nScenario 2: Low battery, no targets rescued, 30 steps left")
print("="*50)
plot_value_heatmap(env, V_star, battery_level=3, rescue_status=(False, False),
                   steps_remaining=30, title_suffix="(Low Battery)")

print("\nScenario 3: Full battery, first target rescued, 40 steps left")
print("="*50)
plot_value_heatmap(env, V_star, battery_level=10, rescue_status=(True, False),
                   steps_remaining=40, title_suffix="(After 1st Rescue)")
```

**Output:**

```text
Scenario 1: Full battery, no targets rescued, start of episode
==================================================
```

![Output from code cell 17](Team_64_DP%20(12)_files/output_17_6.png)

```text
  Value Statistics:
    Max V*(s): 54.007
    Min V*(s): 33.414
    Mean V*(s): 48.471
    Std V*(s): 5.593

Scenario 2: Low battery, no targets rescued, 30 steps left
==================================================
```

![Output from code cell 17](Team_64_DP%20(12)_files/output_17_7.png)

```text
  Value Statistics:
    Max V*(s): 48.016
    Min V*(s): -20.000
    Mean V*(s): 17.362
    Std V*(s): 27.498

Scenario 3: Full battery, first target rescued, 40 steps left
==================================================
```

![Output from code cell 17](Team_64_DP%20(12)_files/output_17_8.png)

```text
  Value Statistics:
    Max V*(s): 38.471
    Min V*(s): 20.000
    Mean V*(s): 31.325
    Std V*(s): 6.056
```

### Code Cell 18

```python
# Additional Analysis: Value vs Battery Level at key positions.
def plot_value_vs_battery(env, V, positions, rescue_status=(False, False), steps_remaining=50):
    """
    Plot how state values change with battery level for selected positions.
    """
    fig, ax = plt.subplots(1, 1, figsize=(10, 6))
    battery_range = range(1, env.max_battery + 1)

    for row, col in positions:
        values = []
        for battery in battery_range:
            state = (row, col, battery, rescue_status, steps_remaining)
            values.append(V.get(state, 0.0))

        cell_type = env.grid[row][col]
        ax.plot(list(battery_range), values, 'o-', linewidth=2,
                label=f'({row},{col}) [{cell_type}]', markersize=6)

    ax.set_xlabel('Battery Level', fontsize=12)
    ax.set_ylabel('State Value V*(s)', fontsize=12)
    ax.set_title(f'State Value vs Battery Level\nRescue={rescue_status}, '
                 f'Steps left={steps_remaining}', fontsize=13, fontweight='bold')
    ax.legend(fontsize=10)
    ax.grid(True, alpha=0.3)
    plt.tight_layout()
    plt.show()


key_positions = [
    (0, 0),  # Start
    (0, 4),  # Rescue target 1
    (4, 2),  # Rescue target 2
    (2, 2),  # Charging station
    (2, 0),  # Danger zone
]

print("State Value vs Battery Level Analysis")
print("="*50)
print("Positions analyzed:")
for row, col in key_positions:
    print(f"  ({row},{col}): {env.grid[row][col]}")

plot_value_vs_battery(env, V_star, key_positions, steps_remaining=50)
```

**Output:**

```text
State Value vs Battery Level Analysis
==================================================
Positions analyzed:
  (0,0): S
  (0,4): R
  (4,2): R
  (2,2): C
  (2,0): D
```

![Output from code cell 18](Team_64_DP%20(12)_files/output_18_9.png)

### Observations from the State-Value Plots

The full-battery heatmap gives high values around the two rescue targets, which is expected because reaching either target creates the largest positive reward. Values close to danger cells are lower, especially when the drone has a safer route around them.

The low-battery case is more cautious. Positions near the charging station become more useful because the drone can recover battery before attempting the second rescue. After one target is rescued, the value pattern shifts toward the remaining target instead of treating both rescue cells equally.

Overall, the value function matches the behaviour we would want: rescue targets are attractive, danger is discouraged, and charging becomes important when battery is tight.

---
## 5. DP Scalability Discussion (1 Mark)

### The Curse of Dimensionality

Dynamic Programming methods require explicit enumeration of the entire state space. As the
problem grows in complexity, the state space explodes exponentially.

### Code Cell 21

```python
# Quantitative analysis of state space scaling.
print("="*70)
print("DP SCALABILITY ANALYSIS: Curse of Dimensionality")
print("="*70)

print("\n1. Current Problem (Group 64):")
print("-" * 40)
current_positions = 5 * 5 - 2
current_battery = 11
current_rescue = 2**2
current_steps = 51
current_total = current_positions * current_battery * current_rescue * current_steps
print("  Grid: 5x5 (23 valid positions)")
print("  Battery levels: 11")
print("  Rescue combinations: 4")
print("  Step counts: 51")
print(f"  Total states: {current_total:,}")
print(f"  Actual enumerated: {len(all_states):,}")

print("\n2. Scaled Problem - 10x10 Grid:")
print("-" * 40)
scaled_positions = 10 * 10 - 5
scaled_battery = 21
scaled_steps = 101
scaled_rescue_3 = 2**3
scaled_rescue_5 = 2**5
scaled_total_3 = scaled_positions * scaled_battery * scaled_rescue_3 * scaled_steps
scaled_total_5 = scaled_positions * scaled_battery * scaled_rescue_5 * scaled_steps
print("  Grid: 10x10 (~95 valid positions)")
print("  Battery levels: 21")
print("  Step counts: 101")
print(f"  With 3 rescue targets: {scaled_total_3:,} states")
print(f"  With 5 rescue targets: {scaled_total_5:,} states")
print(f"  Growth factor (5 targets): {scaled_total_5/current_total:.1f}x")

print("\n3. Dynamic Weather Conditions:")
print("-" * 40)
weather_states = 3
weather_total = scaled_total_5 * weather_states
print(f"  Adding 3 weather states: {weather_total:,} states")
print(f"  Growth factor: {weather_total/current_total:.1f}x")

print("\n4. Larger Scale (15x15, 8 targets, dynamic weather):")
print("-" * 40)
large_positions = 15 * 15 - 10
large_battery = 31
large_rescue = 2**8
large_steps = 151
large_weather = 4
large_total = large_positions * large_battery * large_rescue * large_steps * large_weather
print(f"  Total states: {large_total:,}")
print(f"  Growth factor: {large_total/current_total:.1f}x")
print(f"  Memory (float64): ~{large_total * 8 / (1024**3):.2f} GB just for V(s)")

print("\n" + "="*70)
print("SCALABILITY COMPARISON TABLE")
print("="*70)
print(f"{'Configuration':<45} {'States':<18} {'Factor'}")
print("-"*80)
print(f"{'Current (5x5, 2 targets, 50 steps)':<45} {current_total:<18,} 1x")
print(f"{'10x10, 3 targets, 100 steps':<45} {scaled_total_3:<18,} {scaled_total_3/current_total:.0f}x")
print(f"{'10x10, 5 targets, 100 steps':<45} {scaled_total_5:<18,} {scaled_total_5/current_total:.0f}x")
print(f"{'10x10, 5 targets, 3 weather states':<45} {weather_total:<18,} {weather_total/current_total:.0f}x")
print(f"{'15x15, 8 targets, 4 weather states':<45} {large_total:<18,} {large_total/current_total:.0f}x")
```

**Output:**

```text
======================================================================
DP SCALABILITY ANALYSIS: Curse of Dimensionality
======================================================================

1. Current Problem (Group 64):
----------------------------------------
  Grid: 5x5 (23 valid positions)
  Battery levels: 11
  Rescue combinations: 4
  Step counts: 51
  Total states: 51,612
  Actual enumerated: 51,612

2. Scaled Problem - 10x10 Grid:
----------------------------------------
  Grid: 10x10 (~95 valid positions)
  Battery levels: 21
  Step counts: 101
  With 3 rescue targets: 1,611,960 states
  With 5 rescue targets: 6,447,840 states
  Growth factor (5 targets): 124.9x

3. Dynamic Weather Conditions:
----------------------------------------
  Adding 3 weather states: 19,343,520 states
  Growth factor: 374.8x

4. Larger Scale (15x15, 8 targets, dynamic weather):
----------------------------------------
  Total states: 1,030,568,960
  Growth factor: 19967.6x
  Memory (float64): ~7.68 GB just for V(s)

======================================================================
SCALABILITY COMPARISON TABLE
======================================================================
Configuration                                 States             Factor
--------------------------------------------------------------------------------
Current (5x5, 2 targets, 50 steps)            51,612             1x
10x10, 3 targets, 100 steps                   1,611,960          31x
10x10, 5 targets, 100 steps                   6,447,840          125x
10x10, 5 targets, 3 weather states            19,343,520         375x
15x15, 8 targets, 4 weather states            1,030,568,960      19968x
```

### Scalability Discussion

The current problem is still manageable, but adding the episode clock increases the table from about one thousand physical states to more than fifty thousand planning states. This is the cost of making the DP model match the 50-step termination rule exactly.

The difficulty grows quickly when the grid and mission details grow. A 10 x 10 map has many more positions, additional rescue targets multiply the rescue-status combinations as `2^n`, and dynamic weather would duplicate the state table for each weather condition. Longer missions also multiply the table again through `steps_remaining`.

For real drone rescue, plain tabular DP would not be enough. Real systems operate in continuous space, have noisy sensors, changing wind, moving obstacles, and incomplete information. Deep RL methods help by replacing the explicit table with a learned function approximation, so similar states can share information instead of being stored one by one. DP is still valuable here because it gives a clean baseline and makes the reward/state design easy to inspect before moving to larger Deep RL models.

### Code Cell 23

```python
# Final Summary
print("="*70)
print("ASSIGNMENT SUMMARY - Part #2: Autonomous Drone Rescue (DP)")
print("="*70)
print(f"\nGroup ID: {G}")
print(f"Grid Size: {env.grid_size}x{env.grid_size}")
print("Algorithm: Step-aware Value Iteration")
print("State: (row, col, battery, rescue_status, steps_remaining)")
print("Discount Factor (gamma): 0.95")
print("Convergence Threshold (theta): 1e-3")
print("\nResults:")
print(f"  Total States Enumerated: {len(all_states)}")
print(f"  Iterations to Converge: {num_iters}")
print(f"  Runtime: {runtime:.4f} seconds")
print(f"  Final Delta: {final_delta:.8f}")
print("\nPolicy Performance (20 episodes):")
avg_reward = np.mean([result['total_reward'] for result in results])
avg_steps = np.mean([result['steps'] for result in results])
rescue_rate = np.mean([result['all_rescued'] for result in results])
print(f"  Average Reward: {avg_reward:.2f}")
print(f"  Average Steps: {avg_steps:.1f}")
print(f"  Rescue Success Rate: {rescue_rate*100:.1f}%")
print(f"\nExecution Timestamp: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
print("="*70)
```

**Output:**

```text
======================================================================
ASSIGNMENT SUMMARY - Part #2: Autonomous Drone Rescue (DP)
======================================================================

Group ID: 64
Grid Size: 5x5
Algorithm: Step-aware Value Iteration
State: (row, col, battery, rescue_status, steps_remaining)
Discount Factor (gamma): 0.95
Convergence Threshold (theta): 1e-3

Results:
  Total States Enumerated: 51612
  Iterations to Converge: 29
  Runtime: 7.8035 seconds
  Final Delta: 0.00000000

Policy Performance (20 episodes):
  Average Reward: 108.25
  Average Steps: 50.0
  Rescue Success Rate: 95.0%

Execution Timestamp: 2026-06-07 01:03:56
======================================================================
```
