# src/xrGame/sight_action.h

> Declares one look order for a creature: which sight type, its payload, and the per-type execution state it accumulates while running.

**Needs** — [`sight_manager_space.h`](sight_manager_space.h.md) · [`control_action.h`](control_action.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`sight_action.cpp`](sight_action.cpp.md) · [`sight_action_inline.h`](sight_action_inline.h.md) · [`sight_control_action.h`](sight_control_action.h.md) · [`sight_manager.cpp`](sight_manager.cpp.md)
**Tier floor** — T2: per-frame aiming state

## Purpose

Declares the surface implemented in [`sight_action.cpp`](sight_action.cpp.md) and
[`sight_action_inline.h`](sight_action_inline.h.md).

## Exported units

- **The class** — a control action (so the selector in
  [`setup_manager_inline.h`](setup_manager_inline.h.md) can run it) carrying a sight type
  and whatever that type needs.
- **Six constructors** — one per way of stating a look order; see the inline twin.
- **`initialize` / `execute` / `finalize`** — the action lifecycle. `execute` dispatches
  on sight type.
- **`on_frame`** — the *every-frame* half, as opposed to `execute` which runs at the
  manager's rate. Only two sight types do anything here.
- **`remove_links`** — told when an entity is destroyed; degrades an object-tracking
  order into a fixed-direction one.
- **`target_reached`** — has the head arrived at the commanded yaw.
- **Speed overrides** — whether this order wants a non-default body or head turn speed,
  and what it wants.
- **Equality** — whether two orders are the same order; see the inline twin, where it is
  load-bearing.
- **Payload setters and getters** — direction/position vector, object to look at,
  remembered perception record.
- **`state_fire_object`** — which of the two aiming sub-states the fire-at-object order is
  in; read by the sight manager to decide what to aim at.

## Notes

The per-type execution state (internal state index, state entry times, the remembered
cover yaw, the two remembered positions, the switch time and the switched flag) is
declared here but is meaningful for only one sight type at a time. It is a union written
as a struct. A rebuild with sum types should make the payload and the running state one
tagged value per sight type; that would also remove the need for the equality method's
per-type switch.
