# 🤖 Multi-Robot Area Coverage Simulation

This repository contains the simulation accompanying my article:

📖 **[Collective Intelligence in Robotics: How Multi-Agent Systems Coordinate Physical Robots — Read on Medium](https://medium.com/@aliusman3230/collective-intelligence-in-robotics-how-multi-agent-systems-coordinate-physical-robots-ff95bedd26ef)**

The experiment explores a simple question:

*If a team of robots must explore an area together, how much does the way they coordinate matter?*

It compares four strategies, ranging from random exploration to centralized map sharing. It also investigates how coordination performs under failures, including lost messages, robot failures, and central server outages.

---

### 📊 What You Get

| Output | Description |
|---|---|
| Terminal output | Steps to reach 95% coverage, completed runs, and messages sent for every strategy and condition |
| `results.json` | All experiment results, including average coverage curves |
| `coverage.png` | Coverage over time with all robots working and with half the robots failing |
| `scaling.png` | Speed and message count as the team grows from 2 to 16 robots |

---

## 🧠 How the Simulation Works

### 🌍 The Environment

- The exploration area is a **40 × 40 grid** of cells.
- **Six robots** start together in the top-left corner, at a random position within the first 4 × 4 cells.
- Each robot can sense a **5 × 5 patch** of cells around itself.
- Time advances in discrete steps.
- At each step, every robot moves one cell up, down, left, or right.
- The objective is to achieve **95% area coverage**.
- Each run stops at a maximum of **1,500 steps**. If the target is not reached, the run is recorded as 1,500 steps and marked as unfinished.

### 🤝 The Four Coordination Strategies

| Strategy | Type | Description |
|---|---|---|
| `random` | Swarm rule | Moves randomly at every step. No memory and no communication. |
| `repulse` | Swarm rule | Moves away from teammates within 8 cells to encourage spatial distribution. With 70% probability, it moves in the repulsion direction; otherwise, it moves randomly. |
| `frontier_local` | Distributed | Maintains its own map of explored cells and moves toward the nearest unseen cell. Nearby teammates' goals are penalized. Robots merge maps when within radio range (10 cells). |
| `frontier_shared` | Centralized | Uses the same logic as `frontier_local`, but all robots read and write to a shared global map on a central server. Every robot can see every teammate's goal. |

### 🎯 How a Frontier Robot Selects Its Next Goal

A robot selects a new goal only when it has no current goal or its current goal has already been explored.

For each unseen cell, it calculates a cost:

```text
cost = distance to the cell (in grid steps)
     + 12 for every teammate whose goal is within 8 cells of that cell
```

The robot selects the cell with the lowest cost.

This penalty discourages multiple robots from exploring the same area.

### 📡 Communication & Message Counting

- **`frontier_local`:** Whenever two robots are within radio range, both send their maps. This counts as 2 messages and occurs at every step they remain in range.
- **`frontier_shared`:** Each working robot uploads to the central server once per step. This counts as 1 message per robot per step.
- Messages are counted only until the team reaches 95% coverage.

⚠️ A server message carries an entire map. Therefore, comparing message counts is more favorable to the centralized server than comparing total data transmitted.

### 🧪 Stress Tests

| Condition | Description |
|---|---|
| 3 robots fail at step 60 | Robots 1–3 stop permanently at step 60. |
| 50% message loss | In `frontier_local`, each map exchange fails with a 50% probability. |
| Radio range 4 | In `frontier_local`, robots must be within 4 cells instead of 10 to share maps. |
| Server down from step 20 | In `frontier_shared`, the server stops at step 20. Each robot then uses its own map and sees teammates' goals only within radio range. |

### 📈 Scaling Experiment

The simulation also evaluates `repulse`, `frontier_local`, and `frontier_shared` under normal conditions with:

- 2 robots
- 4 robots
- 8 robots
- 12 robots
- 16 robots

Each configuration uses 10 runs.

### 🔁 Repeatability

Every experiment uses a fixed random seed:

- Main table: seeds 0–14
- Scaling experiment: seeds 0–9

Running the script again produces exactly the same results.

---

## 📊 Expected Results

With the default settings, the following results should be obtained (mean ± standard deviation over 15 runs):

| Strategy | Condition | Steps to 95% | Finished | Messages |
|---|---|---:|---:|---:|
| `random` | Normal | 1448 ± 97 | 5/15 | 0 |
| `random` | 3 robots fail at step 60 | 1496 ± 14 | 1/15 | 0 |
| `repulse` | Normal | 1124 ± 305 | 12/15 | 0 |
| `repulse` | 3 robots fail at step 60 | 1500 ± 0 | 0/15 | 0 |
| `frontier_local` | Normal | 93 ± 5 | 15/15 | 1043 |
| `frontier_local` | 50% message loss | 98 ± 6 | 15/15 | 1047 |
| `frontier_local` | Radio range 4 | 170 ± 17 | 15/15 | 386 |
| `frontier_local` | 3 robots fail at step 60 | 115 ± 11 | 15/15 | 851 |
| `frontier_shared` | Normal | 79 ± 4 | 15/15 | 473 |
| `frontier_shared` | 3 robots fail at step 60 | 92 ± 5 | 15/15 | 456 |
| `frontier_shared` | Server down from step 20 | 244 ± 22 | 15/15 | 120 |

### 🔍 Main Findings

1. 🧠 Robots that remember and share explored information finish more than 10× faster than robots that wander randomly.
2. 🖥️ The centralized server team is the fastest, but a server failure makes it approximately 3× slower and less efficient than peer-sharing coordination.
3. 📡 The peer-sharing team barely slows down when 50% of its messages are lost.
4. 📈 Beyond approximately 8 robots, adding more robots provides diminishing returns, while peer-to-peer message counts continue to grow rapidly.

---

## 🛠️ Changing the Experiment

The main simulation parameters are defined at the top of `coverage_sim.py`:

```python
GRID = 40            # Arena size (GRID x GRID cells)
SENSE = 2            # Sensing radius: 2 means a 5x5 patch
MAX_STEPS = 1500     # Maximum steps per run
TARGET = 0.95        # Coverage goal
CLAIM_PENALTY = 12.0 # How strongly robots avoid teammates' goals
CLAIM_RADIUS = 8     # Distance considered "near" a teammate's goal
```

You can also call `run()` directly to experiment with custom settings:

```python
from coverage_sim import run

r = run(
    "frontier_local",
    seed=0,
    n_robots=10,
    radio_range=6,
    msg_loss=0.3
)

print(r["t"], r["finished"], r["msgs"])
```

The `run()` function accepts:

- `method`
- `seed`
- `n_robots`
- `radio_range`
- `msg_loss`
- `robot_fail_step`
- `n_failed`
- `server_fail_step`

---

## ⚠️ Limitations

This is a deliberately simplified model intended to illustrate coordination trade-offs rather than simulate real robots in full detail.

- Robots move perfectly on a grid. Sensor noise, localization errors, and wheel slip are not modeled.
- Robots can occupy the same cell, so collisions are not modeled.
- Radio range is a simple distance cutoff. Walls and interference are not modeled.
- Message size and communication delay are ignored. Only the number of messages is counted.

---

## 📚 Citation

If you use this simulation or build upon this work, please cite the accompanying article:

**Collective Intelligence in Robotics: How Multi-Agent Systems Coordinate Physical Robots** (2026).

📖 [Read the full article on Medium](https://medium.com/@aliusman3230/collective-intelligence-in-robotics-how-multi-agent-systems-coordinate-physical-robots-ff95bedd26ef)


---

⭐ If you find this simulation useful for research or learning about multi-robot coordination, feel free to explore the code and experiment with the strategies.
