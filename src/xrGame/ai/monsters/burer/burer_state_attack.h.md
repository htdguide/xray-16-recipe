# src/xrGame/ai/monsters/burer/burer_state_attack.h

> Declares the burer's attack tree: the arbiter that picks between gravity, telekinesis, shield, anti-aim and plain repositioning.

**Needs** — [`state.h`](../state.h.md) · [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md)
**Used by** — [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) · [`burer_state_manager.cpp`](burer_state_manager.cpp.md)
**Tier floor** — T2: arbitration over abilities that take the creature over

## Purpose

Declares the surface implemented in [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md).

## `BurerAttackState`

A composite state holding the arbitration bookkeeping:

```text
RECORD BurerAttackState
  waiting_for_substate_end : bool   # a committed sub-attack is running to completion
  lost_health_since_check  : bool   # health dropped by more than the threshold
  allow_anti_aim           : bool   # the arbitration latch
  last_health              : real
  next_runaway_allowed_tick: int
```

It overrides `initialize`, `execute`, `finalize`, `critical_finalize` and the control-arbitration hook.
