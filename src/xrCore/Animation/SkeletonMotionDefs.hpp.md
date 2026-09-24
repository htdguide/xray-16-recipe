# src/xrCore/Animation/SkeletonMotionDefs.hpp

> The five constants the runtime animation format is built on: the sample rate, the quantizer scale, and the partition count.

**Needs** — [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md)
**Used by** — [`KinematicAnimatedDefs.h`](../../Layers/xrRender/KinematicAnimatedDefs.h.md) · [`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) · [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md)
**Tier floor** — T1: these are the constants a frozen byte layout is defined in terms of.

## Purpose

Four numbers that a rebuild cannot choose, isolated so that nothing can quietly disagree about them. Everything in the runtime animation format is expressed relative to these.

## Constants

```text
partitions    = 4        # a skeleton is divided into at most four independently
                         # animated parts (typically: legs, torso, head, and one
                         # spare). Blending happens per partition.

sample_rate   = 30       # animation is RESAMPLED to exactly 30 samples per
                         # second when it is converted from the authored form.
                         # Frozen: a motion's LENGTH is derived by dividing its
                         # sample count by this, so changing it retimes every
                         # shipped animation.

sample_period = 1 / 30   # the interval between samples, in seconds

end_epsilon   = sample_period + a floating-point epsilon
                         # how close to a motion's end counts as "at the end".
                         # One sample plus slack: a blend that is within one
                         # sample of the last key has no further key to
                         # interpolate toward, so it must be treated as finished
                         # rather than reading past the array.

quantizer     = 32767    # rotation components and 16-bit translation
                         # components are stored as signed values scaled by
                         # this, so the full signed 16-bit range maps to
                         # [-1, +1] with the extreme negative value unused.
```

**Invariants** — the quantizer is 32767, not 32768. The stored value is a signed 16-bit integer and the scale is chosen so that +1 maps exactly to the largest positive value; the most negative representable value is never produced. Using 32768 would make +1 overflow to negative. Its reciprocal is precomputed so the decode is a multiply.

**Notes** — the sample rate being fixed rather than stored is the strongest simplification in the runtime format: a motion is a plain array indexed by `time * 30`, with no per-motion timing data at all. It is also the reason an authored motion at a different rate is resampled at build time and cannot round-trip.

Four partitions is a fixed array, not a limit that grew. The blending machinery indexes it directly and a fifth would require touching every consumer.
