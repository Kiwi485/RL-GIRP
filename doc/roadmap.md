# Implementation Roadmap

This roadmap is a distilled version of the project specification and is intended to keep the development path clear and executable.

## Phase 1: Environment connection

- Set up the GIRP socket server
- Connect Python as the client/server bridge
- Use a simple JSON protocol for the first version
- Validate the end-to-end loop:
  - Reset
  - State
  - Random action
  - New state
  - Reward
  - Done / success
  - Reset

## Phase 2: Observation extraction

- Capture player state: position, velocity, angle, angular velocity
- Capture hand state and relative positions
- Detect hold ownership and nearby hold candidates
- Normalize observation values before feeding them to the model
- Keep the observation physics-aware instead of purely position-based

## Phase 3: Action execution

- Define a hold-based action space rather than keyboard-letter mapping
- Choose between discrete and multi-discrete action definitions
- Add no-op support for waiting, swing timing, and momentum use
- Add release actions for left and right hands
- Keep the action system simple enough to debug and mask cleanly

## Phase 4: RL training

- Build the Gymnasium-style environment wrapper
- Train a baseline PPO model
- Add reward logic, episode handling, and reset logic
- Log metrics with TensorBoard and CSV output
- Save regular checkpoints and the best model

## Phase 5: Evaluation

Measure these metrics before increasing complexity:

- Success rate
- Completion time
- Average height
- Episode length
- Death rate
- Learned strategy quality

## Phase 6: Training speed optimization

Prioritize in this order:

1. Rendering off
2. Sound and UI off
3. Faster-than-real-time physics
4. Frame skip
5. Multiple environments
6. Action masking
7. Early termination
8. Curriculum learning
9. Observation simplification
10. Binary socket protocol
11. Headless physics backend

## Phase 7: Fixed-map stabilization

- Train until the agent can reliably reach the goal on the original map
- Set a clear success threshold before moving to randomization
- Use evaluation runs to confirm the agent is not only memorizing a one-off route

## Phase 8: Randomization and generalization

After fixed-map success:

- Stage 1: original fixed map
- Stage 2: letter seed, initial angle, initial velocity
- Stage 3: small hold position jitter
- Stage 4: seed-based procedural map
- Stage 5: unseen seed evaluation

The main goal is not just randomization itself, but ensuring the model can generalize across valid, solvable layouts.

## Development philosophy

Simple version first

↓

Verify learning

↓

Analyze failure

↓

Increase complexity only when the current version works

This keeps the project manageable and avoids expensive rework from overengineering too early.

## Open work items

- Final action space definition
- Action mask rules
- Reward function details
- Reset strategy
- Done / early termination conditions
- Observation hold ordering, padding, and normalization
- Socket protocol format
- Frame skip tuning
- Multi-environment scaling
- Randomization parameters and success thresholds

## Recommendation

This roadmap is a good structure for the repo because it keeps the implementation path visible without burying the README in a very long specification. The main README should act as a project overview, while the detailed planning belongs in the specification files and this roadmap file.
