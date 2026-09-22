# GIRP Reinforcement Learning Project

This repository captures the design for turning the existing GIRP game into a Python-based reinforcement learning environment.

## Project goal

The goal is to keep the original GIRP game logic and physics while exposing observations and actions through a Python RL interface. The pipeline is intentionally simple at first:

```text
GIRP / ActionScript / Box2D
        ↓
Observation extraction
        ↓
TCP socket bridge
        ↓
Gymnasium environment
        ↓
PPO / MaskablePPO
        ↓
Action execution back into GIRP
```

We begin with a fixed map and only add randomization after learning is stable.

## Documentation

- Full project specification: [doc/GIRP_RL_Project_Spec_v1.md](doc/GIRP_RL_Project_Spec_v1.md)
- Chinese project spec: [doc/GIRP_RL_Project_Spec_v1_CH.md](doc/GIRP_RL_Project_Spec_v1_CH.md)
- Implementation roadmap: [doc/roadmap.md](doc/roadmap.md)

## Core roadmap

1. Build the GIRP ↔ Python socket connection
2. Extract physics-aware observations
3. Execute hold-based actions from Python
4. Add reward, done, and reset logic
5. Validate with random-agent and PPO baseline runs
6. Optimize training speed and parallel envs
7. Scale to randomized maps and generalization evaluation

## Design principles

- Start with the simplest working version
- Validate learning before adding complexity
- Prefer hold-based actions over direct keyboard mapping
- Use physics-aware observations so the agent can exploit momentum and swing
- Keep the fixed map stable before introducing randomization

## Project status

The repo currently contains the detailed specification and planning notes. The roadmap was split into its own file so the main README stays readable and the project plan remains easy to track.
