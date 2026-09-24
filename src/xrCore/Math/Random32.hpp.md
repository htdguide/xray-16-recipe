# src/xrCore/Math/Random32.hpp

> A one-word linear congruential generator whose range reduction is a widening multiply rather than a remainder.

**Needs** — [`xrCore.h`](../xrCore.h.md)
**Used by** — [`operator_condition_inline.h`](../../xrAICore/Components/operator_condition_inline.h.md) · [`trivial_encryptor.cpp`](../Crypto/trivial_encryptor.cpp.md) · [`Random.cpp`](../../xrGame/Random.cpp.md) · [`Random.hpp`](../../xrGame/Random.hpp.md)
**Tier floor** — T2: the sequence depends on 32-bit unsigned wraparound and on a 64-bit intermediate product, both of which any tier can express; nothing here touches a device.

## Purpose

A deterministic, explicitly-seeded random source that a caller can own privately, as opposed to the process-wide generator in [`_random.h`](../_random.h.md). Because the seed is a field and the step is pure, a caller can snapshot and restore a stream — which is what the off-screen simulation needs, since it must replay the same decisions after a save/load round trip.

This is a *reproducibility* device, not a statistical one. A rebuild must keep the exact recurrence and the exact reduction: a saved game restores the seed, and the entities that hatch from it must make the same choices they made before.

## State

```text
RECORD Random32
  seed : int (32-bit, wraps)
```

## `Random32`

**Contract** — the seed is readable and writable directly; there is no hidden initialization, so a freshly constructed generator has an indeterminate seed and the caller must set one. Drawing a value advances the seed and returns a number in `[0, range)`. A range of zero yields zero. Pure aside from the seed; no allocation; not safe to share between threads, which is the point — each owner keeps its own.

```text
FUNCTION draw(range: int) -> int
  seed <- (0x08088405 * seed + 1)        # 32-bit, wrapping
  RETURN high 32 bits of (seed as 64-bit * range as 64-bit)
```

**Notes** — the multiplier `0x08088405` and the increment `1` are the Borland/Turbo C constants, inherited rather than chosen; the only property that matters now is that they are the ones the shipped saves were produced with.

The reduction is the interesting decision. Taking a remainder by `range` would bias toward small values and would cost a division; multiplying the full word by `range` and keeping the upper half maps the word's range onto `[0, range)` proportionally, costs one widening multiply, and — crucially — uses the *high* bits of the generator, which in a linear congruential sequence are far better distributed than the low ones. A rebuild that reaches for a modulo will produce a different sequence and a subtly worse one.
