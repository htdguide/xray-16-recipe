# src/xrGame/ai/monsters/group_states/group_state_attack_run.h

> Declares the pack charge state, implemented in
> [`group_state_attack_run_inline.h`](group_state_attack_run_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`group_state_attack_run_inline.h`](group_state_attack_run_inline.h.md)
**Used by** — [`group_state_attack_inline.h`](group_state_attack_inline.h.md) · [`group_state_attack_run_inline.h`](group_state_attack_run_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the leaf state a pack creature runs when it has committed to closing with an enemy. Split
from its body only because C++ splits templates that way.

## `CStateGroupAttackRun`

- **enter** — draw this charge's random interception offset and encirclement window, and read the
  squad's assigned approach direction
- **execute** — recompute the lead point every tick and hand it to the path builder
- **leave** (clean and forced) — stop extrapolating the path
- **is_finished** — inside melee range
- **may_start** — outside melee range

Its private state is three independent random/timing groups — interception, enemy-velocity
prediction, and encirclement — described as a record in the implementation twin.
