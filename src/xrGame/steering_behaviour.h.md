# src/xrGame/steering_behaviour.h

> Declares the six steering forces, the supplier interface each takes its inputs through,
> and the accumulator.

**Needs** — [`steering_behaviour.cpp`](steering_behaviour.cpp.md)
**Used by** — [`ai_monster_squad.cpp`](ai/monsters/ai_monster_squad.cpp.md) · [`ai_monster_squad.h`](ai/monsters/ai_monster_squad.h.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_physic.cpp`](movement_manager_physic.cpp.md) · [`steering_behaviour.cpp`](steering_behaviour.cpp.md)
**Tier floor** — T2: an interface family over vector state.

## Purpose

Declares the surface implemented in
[`steering_behaviour.cpp`](steering_behaviour.cpp.md). One structural decision here is not
in the implementation and must be stated: **each behaviour's inputs are supplied by an object
the caller implements, not pushed in by the caller each frame.**

A behaviour is constructed with a *supplier* which it owns. Once per frame the accumulator
asks each supplier to refresh itself; the supplier reads whatever it needs out of the game —
the object's position, its target's position, its neighbours — writes those into the fields
the behaviour reads, and returns whether the behaviour is still meaningful. Returning false
retires the behaviour and destroys it.

That inversion is what keeps this file free of any game type. A behaviour never knows what a
helicopter is; the supplier does. A rebuild can invert it back — have the caller push a
struct of inputs each frame — and lose only the automatic retirement, which then has to be
expressed some other way.

The supplier is also where two behaviours demand more than data. `containment` requires its
supplier to answer a geometry query — given a probe direction, is there an obstacle, where,
and with what surface normal — so the behaviour never touches the collision database itself.
`grouping` requires its supplier to enumerate neighbours through a start/next/done walk, so
neighbours can be streamed from a spatial query without building a list.

## Exported units

- `base` — the common shape: an enabled flag, a factor triple, a minimum-distance clamp, the
  supplier, and the shared distance falloff. Owns and destroys its supplier.
- `evade` — pull toward a destination while within a maximum range. Supplies a random
  direction when already at the destination.
- `pursue` — close on a destination, with an arrival radius and a braking radius; inside the
  braking radius the acceleration is solved for the wanted arrival speed rather than
  scaled by distance.
- `restrictor` — a leash to an anchor point and a maximum allowed range.
- `wander` — a random-walk heading offset within a chosen coordinate plane, blended back
  toward the current heading by a conservativeness coefficient. The only behaviour with
  state of its own between frames.
- `containment` — probe-based obstacle avoidance, contributing a braking thrust and a
  sideways turn per hit.
- `grouping` — cohesion toward the neighbours' centre plus separation from each close
  neighbour, with separate factor triples for the two halves.
- `manager` — owns a set of behaviours, refreshes them, retires the ones that report
  themselves finished (deferred by one frame), and returns the unweighted sum of the
  enabled ones' accelerations.
- `random_vec` — a uniformly-signed random direction, the default tie-break for the two
  behaviours that can be asked for a direction with no information.

## Notes

The file carries its author's own note that it uses the standard library's containers rather
than the engine's, unlike everything around it. That is incidental — but it does record that
this module arrived later and separately from the rest of the movement code, which is also
why a second, unrelated `base` type shares its namespace; see
[`steering_behaviour_base.h`](steering_behaviour_base.h.md).
