# src/xrGame/sight_control_action.h

> Wraps a look order with the two things the action selector needs from it: a selection weight and a minimum time it must stay chosen.

**Needs** — [`sight_action.h`](sight_action.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md)
**Used by** — [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`sight_control_action_inline.h`](sight_control_action_inline.h.md) · [`sight_manager.h`](sight_manager.h.md)
**Tier floor** — T3: a pair of fields and a clock comparison

## Purpose

The sight manager is an instance of the generic action-selection manager in
[`setup_manager.h`](setup_manager.h.md), which requires of its action type only that it can
report a weight and say whether it has run long enough to be replaced. A bare look order
has neither. This type adds them, and forwards the queries the manager makes of the order
itself (sight type, torso flag, payload) so the manager never has to reach past the
wrapper.

It is a separate file because the *order* and the *scheduling of orders* are separate
concerns; a rebuild that does not use a generic selector can fold the two fields onto the
order and delete this file.

## Exported units

- **The class** — a look order plus a weight and an inertia interval; bodies in
  [`sight_control_action_inline.h`](sight_control_action_inline.h.md).
- **`weight`** — the selector's preference for this order.
- **`completed`** — whether the inertia interval has elapsed since the order started.
- **`use_torso_look`, `sight_type`, `vector3d`, `object`** — forwards to the wrapped
  order's payload.

## Notes

In practice the sight manager only ever holds one order at a time and constructs it with
a weight of one and an effectively infinite inertia (see
[`sight_manager.cpp`](sight_manager.cpp.md)), so the selection machinery this type exists
to satisfy is never exercised for sight. The generality is inherited from the movement
side, which does use it. A rebuild may keep the single-order model and treat both fields
as constants — but then `completed` must still be *false*, not true, or a re-issued
identical order would restart the head turn every tick.
