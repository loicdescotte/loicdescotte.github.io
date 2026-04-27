---
layout: post
title: Evolving Platform Game Agents with a Genetic Algorithm in Scala 3
tags:
 - Scala
 - Scala 3
 - Genetic Algorithm
 - Neural Network
 - FP
 - Java2D
---

# Evolving Platform Game Agents with a Genetic Algorithm in Scala 3

## Introduction

This project explores the following idea: rather than manually programming a player agent's intelligence, we let it emerge through natural selection.
You can find the code [here on Github](https://github.com/loicdescotte/genetic-platformer).

Concretely, a population of 50 agents attempts to traverse a procedurally generated platform level. Each agent is driven by a neural network whose parameters (weights and biases) constitute its "genome". Agents that progress the furthest survive, reproduce, and pass their genes to the next generation. Those that fall into holes or get hit by an enemy are eliminated.

After a few generations, behaviors emerge: agents learn to jump at the right moment, dodge enemies, and adjust their trajectory based on obstacles. No explicit rules teach them these behaviors — everything comes from selection pressure.

The whole thing is written in **Scala 3**, uses **Java2D** for rendering, and runs entirely locally (no GPU, no ML framework). The emphasis is on functional purity: no shared mutable state, no hidden side effects.

---

## Data Model

### Purely Functional Approach

All game logic is **pure and immutable**. Each frame produces a new `GameState` rather than modifying the old one. Randomness is passed explicitly as a parameter — never via global state.

This constraint has concrete benefits: you can replay any frame, compare two states, or evaluate 50 individuals in parallel without risk of interference.

### Coordinate System

```
(0,0) ──────────────────────► X  (positive rightward)
  │
  │   [tiles] [player] [enemies]
  │
  ▼
  Y  (positive downward — Java2D screen convention)
```

The Y axis points **downward**, which is the standard Java2D convention. A jump therefore corresponds to a **negative** Y velocity.

### Main Types

**`Vec2`** — 2D vector for positions and velocities, in pixels:

```scala
case class Vec2(x: Double, y: Double):
  def +(v: Vec2): Vec2   = Vec2(x + v.x, y + v.y)
  def -(v: Vec2): Vec2   = Vec2(x - v.x, y - v.y)
  def *(s: Double): Vec2 = Vec2(x * s, y * s)
```

**`Tile`** — two-case ADT representing a cell in the world grid:

```scala
enum Tile:
  case Air    // empty space, traversable
  case Ground // solid floor or platform
```

**`World`** — immutable tile grid, indexed by `tiles(col)(row)`:

```scala
case class World(tiles: Vector[Vector[Tile]], width: Int, height: Int)
```

**`Player`** — player state at a given instant:

```scala
case class Player(
  position: Vec2,    // top-left corner of the hitbox (pixels)
  velocity: Vec2,    // current velocity (pixels/frame)
  onGround: Boolean, // resting on a solid surface
  alive:    Boolean, // false if dead (hole or enemy)
  maxX:     Double   // maximum X position reached — measures progress
)
```

**`Enemy`** — enemy patrolling between two fixed bounds:

```scala
case class Enemy(
  position:    Vec2,
  velocity:    Vec2,
  direction:   Direction,
  patrolLeft:  Double,
  patrolRight: Double
)
```

**`Actions`** — neural network outputs, decoded as booleans:

```scala
case class Actions(moveLeft: Boolean, moveRight: Boolean, jump: Boolean)
```

Multiple actions can be active simultaneously (e.g. `moveRight + jump`).

**`GameState`** — the complete game state, passed frame by frame:

```scala
case class GameState(
  world:   World,
  player:  Player,
  enemies: Vector[Enemy],
  frame:   Int,
  over:    Boolean
)
```

The fitness function is defined directly on `GameState`:

```scala
def fitness: Double =
  val distance   = player.maxX / GameConfig.TileSize
  val speedBonus = 1.0 - frame.toDouble / GameConfig.MaxFrames
  distance + speedBonus
```

---

## Neural Network

### Architecture

The network is a two-layer feedforward multilayer perceptron:

```
Sensors (11 inputs)
     ↓
Hidden layer: 16 neurons, ReLU activation
     ↓
Output layer: 3 neurons, Sigmoid activation
     ↓
Actions: moveLeft | moveRight | jump
```

Each neuron computes `activation(Σ(input_i × weight_i) + bias)`. Forward propagation is a **pure** operation — no state modified.

### Genome: a flat sequence of 243 real numbers

Each individual is represented by a `Genome`, i.e. a `Vector[Double]` of 243 values. These genes encode all network parameters in the following order:

| Segment | Content | Size |
|---|---|---|
| 1 | Layer 1 weights (11 × 16 connections) | 176 |
| 2 | Layer 1 biases (16 neurons) | 16 |
| 3 | Layer 2 weights (16 × 3 connections) | 48 |
| 4 | Layer 2 biases (3 neurons) | 3 |
| **Total** | | **243** |

This flat representation allows crossover and mutation to be applied uniformly, without knowledge of the network's internal structure.

### `Genome.decode()`

The `decode` function reconstructs a `Network` from a `Genome` and a `NetworkConfig`. It traverses the genes sequentially: for each layer, it reads the `inSize × outSize` weights first, then the `outSize` biases.

```scala
Genome (243 genes)
  └─ Genome.decode() ──► Network (weights + biases)
                              │
GameState ──► Agent.observe() ──► [11 floats] ──► Network.forward() ──► [3 floats]
                                                                              │
                                                                    Agent.act() ──► Actions
                                                                                        │
                                                              Engine.step(state, actions) ──► next GameState
```

---

## Sensors: the 11 network inputs

Each frame, `Agent.observe()` transforms the raw game state into a vector of 11 normalized numerical values that the network can process.

| Index | Feature | Range | Description |
|---|---|---|---|
| 0 | Ground present 1 tile ahead | 0 or 1 | Detects immediate hole edges |
| 1 | Ground present 2 tiles ahead | 0 or 1 | |
| 2 | Ground present 3 tiles ahead | 0 or 1 | |
| 3 | Ground present 4 tiles ahead | 0 or 1 | |
| 4 | Distance to next hole | [0, 1] | Normalized over 8 tiles; 0 = jump now, 1 = no hole in sight |
| 5 | Horizontal distance to enemy ahead | [0, 1] | Normalized over 12 tiles; 1 = no enemy |
| 6 | Vertical distance to enemy ahead | [-1, 1] | Normalized over 4 tiles; negative = enemy above |
| 7 | Player vertical velocity | [-1, 1] | Normalized by `MaxFallSpeed`; negative = rising |
| 8 | Player on ground | 0 or 1 | `onGround` flag |
| 9 | Vertical clearance ahead | [0, 1] | 0 = ceiling right above, 1 = full jump height free |
| 10 | Enemy direction | -1, 0 or +1 | +1 = approaching, -1 = moving away, 0 = no enemy |

Sensors 0 to 3 form a "ground scan": they allow the network to detect a hole before falling into it. Sensor 4 refines this signal with pixel precision. Sensor 9 prevents the agent from jumping into a low ceiling — information absent from the first version of the project, whose addition noticeably improved behavior in platform areas.

### Actions

The 3 Sigmoid outputs of the network produce values in [0, 1]. A threshold of 0.5 converts them to booleans:

```
outputs(0) > 0.5  →  moveLeft  = true
outputs(1) > 0.5  →  moveRight = true
outputs(2) > 0.5  →  jump      = true
```

---

## Genetic Algorithm

### Representation

Each individual is a `(Genome, fitness)` pair. The genome is a `Vector[Double]` of 243 values — a direct encoding of all network weights and biases. Evaluation consists of decoding the genome into a network, running the agent until death or a 2500-frame timeout, and recording the score.

### Initial population: intentionally bad agents

Generation 0 is populated with "naive" individuals built via `Genome.naive()`. The idea is to start with stupid but differentiated behaviors, so that selection has material to work with from the very first crossovers.

Three archetypes are generated randomly:

- **Rusher (50%)**: output biases oriented to spam `moveRight` and suppress `jump`. The agent runs right and invariably falls into the first hole. This is the most "visible" behavior in generation 0.
- **Clumsy jumper (25%)**: `moveRight` encouraged, `jump` permanent. The agent moves forward while jumping continuously — it misses landings and jumps at the wrong moments.
- **Erratic (25%)**: heavily saturated weights (σ=3.0), chaotic behavior. It oscillates, retreats, or stands still.

In all cases, the general weights use σ=1.5 (vs σ=0.5 for a standard random network), which saturates activations and prevents the network from correctly processing sensor inputs. Evolution must therefore learn both to calibrate weights and to build jumping strategies.

### Fitness function

```
fitness = maxX / TileSize + (1 - frame / MaxFrames)
```

The main component is the **distance in tiles** reached. The **speed bonus** is 1.0 if the agent finishes instantly, 0.0 if it hits the 2500-frame timeout. Two agents reaching the same distance are therefore ranked by speed. MaxFrames corresponds to approximately 42 seconds at 60 fps.

### Tournament selection (k=3)

3 individuals are drawn randomly from the population and the best is kept. This operation is repeated to select each parent. Selection pressure is moderate: a weak individual always has a non-zero probability of being selected if it faces two even weaker opponents.

### BLX-α crossover (α=0.3)

Each child gene is a random interpolation of the two parental genes, slightly extrapolated beyond their interval:

```
lo  = min(g1, g2) - α × |g2 - g1|
hi  = max(g1, g2) + α × |g2 - g1|
child_gene = lo + u × (hi - lo),  u ~ Uniform[0, 1]
```

With α=0.3, the child can slightly exceed parental values. This crossover preserves network inter-layer coherence better than single-point crossover, where a cut in the middle of the genome can break dependencies between weights of adjacent layers.

### Gaussian mutation

Each gene has a 10% probability of being perturbed by Gaussian noise N(0, σ=0.4). This introduces diversity without destroying good solutions.

### Elitism

The 5 best individuals of each generation pass **unchanged** to the next. This guarantees that the best solutions are never lost to mutation.

### Stagnation detection

If the best fitness remains identical for 8 consecutive generations, the mutation rate increases from 10% to 15% and σ from 0.4 to 0.55. The following message is logged:

```
[GEN 14] Stagnation detected — mutation boosted (rate=0.15, strength=0.55)
```

This mechanism allows escaping local optima without permanently increasing mutation.

---

## Simulation Loop

The simulation is a two-phase state machine, represented by the `SimState` ADT:

```scala
enum SimState:
  case Evaluating(population, remaining, evaluated)
  case Showcasing(population, gameState, network, framesDone)
```

### Phase 1: Evaluating

No rendering. 8 individuals are evaluated per Swing tick. For each individual, its genome is decoded into a network, then `Engine.step()` is looped until `state.over`, calling `Agent.act()` at each iteration. This loop takes less than 2 ms per individual on a modern CPU — all 50 individuals of a generation are processed in a fraction of a second.

Once all individuals are evaluated, fitness scores are computed, the population is sorted, and the simulation moves to the Showcasing phase.

### Phase 2: Showcasing

The best individual of the generation plays in real time at 60 fps, for a maximum of 2500 frames. Java2D rendering is active: you can observe the evolved behavior, watch the character jump over holes, dodge enemies, and accelerate rightward.

![Genetic algorithm — agent showcasing in the Java2D renderer](/images/genetic-sc.png)

At the end of the showcase (death or timeout), `Evolution.evolve()` is called to produce the next generation, and the cycle restarts in the Evaluating phase.

It is also possible to skip the showcase manually via `skipShowcase()`, which immediately triggers evolution.

---

## World Generation

The level is generated **once** at startup with the fixed seed `42L`. All individuals across all generations play on the same level, which guarantees a fair fitness comparison.

Generation proceeds in four passes:

**Pass 1 — Base floor:** rows `FloorRow` and `FloorRow+1` are filled with `Ground` across the full world width (150 tiles).

**Pass 2 — Holes:** from left to right, holes 2 to 3 tiles wide are dug with increasing probability (from 20% to 40% depending on progress through the level). A minimum spacing of 5 to 10 solid tiles separates two holes. Holes are always 2 or 3 tiles wide — crossable with good jump timing.

**Pass 3 — Floating platforms:** platforms 4 to 7 tiles wide are placed 3 to 5 tiles above the floor. A repair pass then fills holes located beneath a low platform, to avoid impossible configurations.

**Pass 4 — Enemies:** enemies are placed on floor segments of at least 6 tiles (with 80% probability) and on platforms of at least 4 tiles (70%). Each enemy patrols back and forth within its segment bounds.

This fixed world, combined with the distance-based fitness, creates a clear and reproducible evolutionary pressure: each generation is evaluated against exactly the same obstacle.
