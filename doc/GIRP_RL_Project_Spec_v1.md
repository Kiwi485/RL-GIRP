# GIRP Reinforcement Learning Project Specification v1.0

## Project Overview

This project aims to train a Reinforcement Learning (RL) agent to control the GIRP climbing game.

The agent should learn:
- Select climbing targets
- Control grabbing and releasing
- Utilize physics-based movement
- Exploit swing momentum
- Learn contraction and relaxation timing
- Complete climbing routes autonomously

Main research question:

Can a Reinforcement Learning agent learn human-like climbing strategies by interacting with the original GIRP physics environment?

---

# System Architecture

```
Python PPO RL Agent
        |
        | TCP Socket + JSON
        |
GIRP Environment
        |
Physics Simulation
```

Python:
- PPO training
- Action selection
- Model saving
- Evaluation
- Logging

GIRP:
- Physics simulation
- State extraction
- Action execution
- Reward calculation
- Episode management

---

# Reset Strategy

Version 1 uses:

Full PlayState Reset

Flow:

Episode End
-> Destroy old state
-> Create new PlayState
-> Reset RL variables
-> Start new episode

Reason:
Prioritize correctness and avoid hidden state leakage.

---

# Observation Space

Version 1:

Upper-body Physics State + 8 Nearby Holds

Includes:

Body:
- Position
- Velocity
- Angle
- Angular Velocity

Hands:
- FREE / ASSIGNED / LOCKED
- Current target
- Position
- Velocity

Arms:
- Joint angle
- Joint velocity
- Contract state

The RL agent does not control hand assignment.
GIRP automatically chooses the hand.

---

# Observation Normalization

All continuous values are normalized to [-1,1].

Includes:
- Position
- Velocity
- Angular velocity
- Joint angles
- Relative hold position

Binary values remain 0/1.

---

# Hold System

Use original GIRP holds.

No new holds are generated.

Nearby Hold Slots:
- 8 total

Hybrid selection:

Slot 0-3:
Nearest candidates

Slot 4-7:
Future candidates

Future candidates consider:

height gain - distance penalty

Actions operate on slots and are mapped internally to original Hold IDs.

---

# Action Space

Version 1:

Discrete(14)

0: No-op

1-8:
Target Nearby Hold Slot 0-7

9:
Release Left Hand

10:
Release Right Hand

11:
Contract

12:
Relax

13:
Cancel Target

---

# Target System

Persistent Target.

A selected target remains active until:

1. Successfully grabbed
2. Replaced by another target
3. Cancel Target action
4. Episode termination

Purpose:

Target -> Swing -> Momentum -> Grab

---

# Contract / Relax

Persistent state.

Contract remains active until Relax is selected.

Allows learning:
- Timing
- Momentum control
- Swing strategies

---

# Action Mask

Use Conservative Action Masking.

Mask only clearly invalid actions:

- Non-existing hold
- Disabled hold
- Release empty hand
- Cancel without target

Do not mask:
- Far holds
- Difficult holds
- Momentum-based targets

---

# Reward Design

Step reward:

reward = current_height - previous_height

Do not initially add:
- Grab reward
- Contract reward
- Swing reward

Success:

Finish Bonus + Completion Time Bonus

Death:
Episode ends without additional penalty.

Timeout:
- No progress timeout
- Hard maximum episode time

---

# Training Architecture

Initial:

1 GIRP Environment + 1 PPO Agent

Future:

Multiple GIRP environments for parallel training.

---

# Communication Protocol

TCP Socket + JSON.

Flow:

GIRP:
State -> Observation

Python:
Observation -> PPO -> Action

GIRP:
Execute Action -> Physics Step -> Reward

---

# Logging

Record:

- Episode reward
- Average height
- Success rate
- Completion time
- Episode length
- Death reason

Visualization:
- Reward curve
- Success curve
- Height progress
- Completion time

---

# Checkpoint

Save:

Regular checkpoints:
Every N training steps

Best model:
- Highest success rate
- Best completion time

---

# Randomization

Not included initially.

After stable climbing:

Possible:
- Random hold positions
- Physics randomization
- Initial state randomization

---

# Implementation Roadmap

Phase 1:
Environment connection

- GIRP socket server
- Python socket client
- JSON protocol

Phase 2:
Observation extraction

- Physics state
- Hand state
- Hold detection
- Normalization

Phase 3:
Action execution

- Target system
- Contract system
- Release system

Phase 4:
RL Training

- PPO
- Training loop
- Logging
- Checkpoints

Phase 5:
Evaluation

Measure:
- Success rate
- Completion time
- Learned strategies

---

# Development Philosophy

Simple Version First

↓

Verify Learning

↓

Analyze Failure

↓

Increase Complexity

Do not increase observation size, action space, or randomization until the current version works.
