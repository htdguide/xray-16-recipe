# src/Layers/xrRender/Animation.h

> Declares the four animation mixing channels and the fixed rule that says how each one combines with the others.

**Needs** — [`KinematicAnimatedDefs.h`](KinematicAnimatedDefs.h.md)
**Used by** — [`Animation.cpp`](Animation.cpp.md) · [`AnimationKeyCalculate.h`](AnimationKeyCalculate.h.md) · [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md) · [`SkeletonAnimated.h`](SkeletonAnimated.h.md)
**Tier floor** — T3: a table of rules and a weight per channel.

## Purpose

Declares the surface implemented in [`Animation.cpp`](Animation.cpp.md). A *channel* is an independent lane of animation playback on one model: lane 0 is the ordinary locomotion animation, and the higher lanes carry things that must play *over* it rather than instead of it. The channel's rule decides whether a second animation in the same lane replaces the first (interpolate) or is layered on top of it (add).

## Exported units

- **`mix_type`** — two ways to combine two poses: `lerp` (weighted interpolation between them) and `add` (apply the second as a delta on top of the first).
- **`channal_rule`** — a pair of mix types for one channel: how animations *within* the channel combine, and how the channel's finished result combines with the channels below it.
- **`channel_def`** — one channel's rule plus its current weight, which is what the per-bone mixing code is actually handed.
- **`channels`** — the per-model channel weights, the immutable rule table, and the two accessors that read them.

**Notes** — The rule table is `static const` and shared by every model in the process: the rules are a property of the *system*, not of a model. Only the weights are per model. A rebuild should keep that split; it is what lets the per-bone mixer read the rule without a pointer chase into the model.
