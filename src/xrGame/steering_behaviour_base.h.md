# src/xrGame/steering_behaviour_base.h

> An abandoned second steering interface, written for rat flocking, never implemented and
> never included.

**Needs** — [`steering_behaviour_base_inline.h`](steering_behaviour_base_inline.h.md)
**Used by** — [`steering_behaviour_alignment.h`](steering_behaviour_alignment.h.md) · [`steering_behaviour_base_inline.h`](steering_behaviour_base_inline.h.md) · [`steering_behaviour_cohesion.h`](steering_behaviour_cohesion.h.md) · [`steering_behaviour_separation.h`](steering_behaviour_separation.h.md)
**Tier floor** — T2: an abstract interface over a creature reference.

## Purpose

This is **dead code**, and the fact is the point of the page.

The file declares an interface with the same name, in the same namespace, as the live
steering interface in [`steering_behaviour.h`](steering_behaviour.h.md), but it is a
different and incompatible design: it is bound to a *rat*, it produces a **direction**
rather than an acceleration, and it has no supplier — an implementation would read the
creature directly. It predates the live file by six months.

No source in the engine includes it. It is listed in the build files, which compile no
translation unit that reaches it, and none of its three declared implementations
([`alignment`](steering_behaviour_alignment.h.md),
[`cohesion`](steering_behaviour_cohesion.h.md),
[`separation`](steering_behaviour_separation.h.md)) has a body anywhere in the tree. The
name collision with the live interface is harmless only because the two are never seen
together.

A rebuild should **not** carry these four files over. What is worth recording is the
intent: rats were to have been flocked with an alignment/cohesion/separation triple, and the
work stopped before any of it was written. Rat movement in the shipped game is done
elsewhere.

## State

```text
RECORD SteeringBehaviourBase          # never instantiated
  creature : reference to Rat         # read-only; set at construction, never used
  enabled  : bool
```

## Exported units

- construction from a rat.
- `direction()` — required of an implementor: this frame's steering direction. Never
  implemented.
- `enabled` (get and set) — see
  [`steering_behaviour_base_inline.h`](steering_behaviour_base_inline.h.md).
