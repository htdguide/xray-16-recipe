# src/xrGame/ai/monsters/zombie/zombie_state_attack_run.h

> Declares the zombie's approach: the only creature-specific leaf state in its behaviour tree.

**Needs** — [`state.h`](../state.h.md) · [`zombie_state_attack_run_inline.h`](zombie_state_attack_run_inline.h.md)
**Used by** — [`zombie_state_attack_run_inline.h`](zombie_state_attack_run_inline.h.md) · [`zombie_state_manager.cpp`](zombie_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`zombie_state_attack_run_inline.h`](zombie_state_attack_run_inline.h.md). It is slotted
into the generic attack composite as its approach phase, replacing the approach every other
creature uses.

## State

```text
RECORD ZombieApproach
  action              : ActionId   # walk forward, or run
  time_action_changed : int        # written by dead code only; see the implementation twin
```

## Exported units

- **the zombie approach state** — entry, per-tick execution, a start condition and a
  completion test both expressed in melee range, and the private gait choice.
