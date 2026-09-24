# src/xrCore/_random.h

> The engine's pseudo-random generator: a 32-bit linear congruential sequence with fixed constants, exposed through a full integer and real vocabulary, plus the one global instance everything shares.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`xrDebug.h`](xrDebug.h.md) · [`_math.cpp`](_math.cpp.md) · [Conformance](../../SYSTEM-REQUIREMENTS.md#6-conformance)

**Used by** — [`vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`_math.cpp`](_math.cpp.md) · [`_vector3d.h`](_vector3d.h.md) · [`vector.h`](vector.h.md) · [`secure_messaging.cpp`](../xrGame/secure_messaging.cpp.md) · [`secure_messaging.h`](../xrGame/secure_messaging.h.md) · [`particle_core.cpp`](../xrParticles/particle_core.cpp.md)

**Tier floor** — T1: the sequence is defined by 32-bit signed multiplication that is *allowed and required* to overflow, and by a shift on the resulting bit pattern. Reproducing it in a tier with arbitrary-precision integers or with trapping overflow means explicitly masking to 32 bits at every step.

## Purpose

Almost every non-deterministic-looking thing in the game — weapon spread, particle
lifetimes, creature wander directions, ambient sound timing, loot rolls — draws from this.
The generator is chosen for two reasons and neither is statistical quality: it is four
arithmetic operations, and its sequence is *exactly reproducible* from a 32-bit seed, which
is what makes the determinism criterion in the conformance list reachable at all.

The interface is deliberately wide. Every place that wants a random number wants it in a
range, and giving each shape a name means no caller writes the modulo or the scaling itself
— which is how a rebuild avoids the classic family of off-by-one and bias bugs scattered
across three hundred call sites.

## State

```text
RECORD RandomGenerator
  state : int (32-bit, signed, wraps)    # the whole of it
```

**Invariants**

- The state is a single 32-bit signed integer and its default value is **1**, not 0. A
  seed of zero is legal and produces a perfectly good sequence; the default is 1 only so
  that an unseeded generator is not all zeros.
- The multiply overflows on essentially every step and the overflow is part of the
  definition. In the original the arithmetic is signed, which is where a rebuild must be
  careful: a tier that traps on signed overflow must do the step in an unsigned 32-bit type
  and reinterpret.
- The state is marked as volatile in the original, which is the closest that tier gets to
  saying "this is touched from more than one thread". It is **not** atomic and the update is
  not a single operation, so concurrent draws can interleave and lose an update. Nothing
  guards it. That is an accepted race: the consequence is a duplicated value, not a
  corrupted state, because any interleaving still leaves a legal 32-bit value behind. A
  rebuild that makes it thread-local gets better numbers and *loses reproducibility*, which
  is the wrong trade for this engine — see the note below.

## The sequence

**Contract** — One step advances the state and yields fifteen bits.

```text
FUNCTION next_raw() -> int      # 0 .. 32767
  state = state * 214013 + 2531011      # 32-bit signed, wraps
  RETURN (state >> 16) AND 0x7fff       # take bits 30..16
```

**Invariants** — The two constants and the shift are the whole specification, and all three
are frozen by the requirement that a seeded run reproduce. The low bits of a linear
congruential sequence have short periods, which is why bits 16 through 30 are taken rather
than the bottom fifteen; the sign bit is masked off so the result is always non-negative.

**Notes** — This is the multiplier and increment of a well-known C library generator of the
period. The range is therefore only fifteen bits, which matters: a caller asking for a
random integer in a range larger than 32768 gets a sequence that can never produce most of
the values in it, and a caller asking for a real gets about four decimal digits of
resolution. Both facts are load-bearing for a rebuild that is tempted to "improve" the
generator — the improvement changes every random-looking thing in the game, and the shipped
balance data was tuned against *this* distribution.

The original carries a commented-out ancestor of the routine that used a different
multiplier and returned the high half of a 64-bit product scaled to a requested range. It
is dead and is not what the game plays against; it is preserved here only so a rebuilder who
finds it in the source knows it is not the live definition.

## `seed`

**Contract** — Replaces the state outright. Total, no validation — any 32-bit value is a
legal seed, including zero.

## The integer draws

**Contract** — Four shapes over `next_raw`, all total, allocation-free, none blocking:

| Shape | Result |
|---|---|
| bare | `0 .. 32767` |
| bounded by `max` | `0 .. max-1`, by remainder; asserts `max` is non-zero |
| bounded by `min, max` | `min .. max-1` |
| symmetric about zero, by `range` | `-range .. range-1` |
| symmetric, offset | the above plus an offset |

**Invariants** — The bounded form is a plain remainder of a fifteen-bit draw. It is
therefore **biased** whenever `max` does not divide 32768, and unusable for any `max` above
32768. Both are true of the original and both are visible in the game's behaviour, so a
rebuild that switches to a rejection-sampling draw is changing observable outcomes. The
assertion on a zero bound is the only validation anywhere in the type.

The symmetric form is asymmetric by one: it spans `-range` through `range - 1`, not through
`range`. That is a consequence of the half-open bounded form and it is what the shipped
tuning was authored against.

## The real draws

**Contract** — The same five shapes in reals, all built on dividing a raw draw by 32767:

| Shape | Result |
|---|---|
| bare | `0 .. 1` inclusive at both ends |
| bounded by `max` | `0 .. max` |
| bounded by `min, max` | `min .. max` |
| symmetric by `range` | `-range .. range` |
| symmetric, offset | the above plus an offset |

**Invariants** — The divisor is 32767, the *largest* draw, not 32768. So the real form is
closed at both ends — it can return exactly 0 and exactly 1 — and its resolution is one part
in 32767. Callers that use it to index an array must clamp; several do not, and the reason
they are correct is that they multiply by a count and truncate, which reaches the count only
on the exact-1 draw.

Unlike the integer forms, the symmetric real form **is** symmetric: it spans `-range`
through `+range` inclusive.

## The global generator

**Contract** — One shared instance that the whole engine draws from unless a caller supplies
its own. Seeded from the monotonic clock during startup — see
[`_math.cpp`](_math.cpp.md) — which means the default behaviour of the engine is *not*
reproducible run to run.

**Notes** — This is the single most important thing on the page for a rebuild. The vector
type's random operations default to this generator, so "random direction" and "random point
in a sphere" are draws from the same shared sequence as everything else. That is what makes
the determinism criterion expressible as "the same level, the same input sequence and the
same seed": there is exactly one sequence, and fixing it fixes the world. It is also what
makes the criterion fragile — any draw taken by a subsystem the replay does not reproduce
(a decorative particle, a debug overlay) shifts the sequence for everything after it.

A rebuild has a genuine design choice here, and it should be made deliberately: keep one
global sequence and accept that every consumer must be replayed, or give each subsystem its
own seeded generator and gain the ability to replay one subsystem in isolation. The
original chose the first, implicitly.
