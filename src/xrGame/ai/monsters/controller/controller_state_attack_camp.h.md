# src/xrGame/ai/monsters/controller/controller_state_attack_camp.h

> Declares the camping sub-state — the controller waiting in cover, sweeping its gaze between two limits — implemented in [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md)
**Used by** — [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md) · [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CStateControlCamp`. Substance is in
[`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md).

## State

```text
RECORD CampState
  angle_from, angle_to : real       # the two limits of the gaze sweep, found at entry
  target_angle         : real       # whichever limit is currently being looked toward
  time_next_updated    : int (ms)   # when the sweep next reverses
```

Exported units: `initialize`, `execute`, `check_completion`, `check_start_conditions`,
`update_target_angle` and an empty `remove_links` — the standard state contract plus one
private helper that flips the sweep.
