# src/xrGame/ai/weighted_random.h

> Declares a sampler that draws from a piecewise-linear distribution over one, two or three points. Included once, used nowhere.

**Needs** — [`weighted_random.cpp`](weighted_random.cpp.md)
**Used by** — [`monster_state_attack_on_run.h`](monsters/states/monster_state_attack_on_run.h.md) · [`weighted_random.cpp`](weighted_random.cpp.md)
**Tier floor** — T3: six numbers and a draw

## Purpose

Declares the surface implemented in [`weighted_random.cpp`](weighted_random.cpp.md).

A designer wants to say "mostly two seconds, sometimes as little as one, rarely as much as
five" without writing a distribution. This type is the answer: name up to three
value-and-weight pairs and the sampler treats them as the vertices of a piecewise-linear
density and draws from it.

**It is dead.** One creature behaviour header includes it and then never names the type. No
object of it is ever constructed and its draw is never called. A rebuilder should keep the
*idea* — an authorable three-point distribution is a good way to express AI timing — and
should not port the implementation, which draws from the C library's global generator (see
the implementation twin).

## State

```text
RECORD WeightedRandom
  a_value, a_weight : real
  b_value, b_weight : real     # weight of -1 means "this point is absent"
  c_value, c_weight : real     # weight of -1 means "this point is absent"
```

**Invariants** — absence is encoded as a weight of exactly minus one, so a caller cannot
legitimately author a negative weight. The number of live points is therefore one, two or
three, decided by which trailing weights are the sentinel.

## Exported units

- **four constructors** — a degenerate one (zero, always), a constant one (one point), a
  two-point one and a three-point one.
- **the constant test** — true when only the first point is live.
- **the draw** — returns one sample.
