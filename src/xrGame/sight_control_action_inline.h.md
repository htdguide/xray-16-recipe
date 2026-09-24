# src/xrGame/sight_control_action_inline.h

> The bodies for the look-order wrapper: construction by copying the order, and the inertia test.

**Needs** — [`sight_control_action.h`](sight_control_action.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field reads and one clock comparison

## Purpose

Carries the bodies for [`sight_control_action.h`](sight_control_action.h.md). One thing
here is load-bearing: what `completed` measures.

## Construction

**Contract** — takes a weight, an inertia interval and a look order; stores the two
numbers and copies the order wholesale, *including its running state*. The copy is made
before the order is initialized, so the copied running state is the freshly-constructed
one.

## `completed`

**Contract** — true once the global clock has advanced past the order's start time by at
least the inertia interval.

```text
FUNCTION completed() -> bool
  RETURN now - start_time >= inertia_time
```

**Invariants** — `start_time` is stamped by the order's own initialization, not by this
wrapper, so the inertia clock starts when the order starts *running*, not when it is
constructed. The sight manager passes the largest representable interval, which makes this
permanently false and pins the chosen order until it is explicitly replaced.

## Accessors

**Contract** — `weight`, `use_torso_look`, `sight_type` and `vector3d` return the
corresponding field. `object` returns the tracked entity and asserts there is one: asking
a non-tracking order for its target is a programming error, because the answer would be
a reference to nothing rather than a wrong answer.
