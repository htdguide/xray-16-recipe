# src/xrCore/PostProcess/PPInfo.cpp

> How two screen-effect parameter blocks combine: scalars add, noise takes the stronger, and colour-mapping textures form a two-slot crossfade.

**Needs** — [`PPInfo.hpp`](PPInfo.hpp.md) · [`xrDebug.h`](../xrDebug.h.md)
**Used by** — [`PPInfo.hpp`](PPInfo.hpp.md)
**Tier floor** — T2: arithmetic on a parameter block.

## Purpose

Several effects are active at once — the actor is bleeding, has taken a drug, and is standing in an anomaly — and each contributes one parameter block. This file decides what "at once" means for each field. The answer is not uniform, and the non-uniformity is the whole content of the file.

## `default construction`

**Contract** — produces the neutral block: no blur, no desaturation, no ghosting, no noise intensity, grain size 1, grain rate 10 per second, a mid-grey contrast pivot, flat grey weights, no additive colour, and no colour mapping. This is the block a frame starts from before any effect is folded in.

**Notes** — the neutral values for the three colours are *not* zero, which is why the block cannot simply be zero-initialized. `color_base` is a pivot, `color_gray` is a weight triple; both mean "no change" at their stated values and mean something very visible at zero.

## `add` — fold one effect into the accumulator

**Contract** — combines another block into this one. Mutates in place; no allocation; total.

```text
FUNCTION add(other)
  # Scalars accumulate: two blurs blur more.
  blur      <- blur + other.blur
  gray      <- gray + other.gray
  duality.h <- duality.h + other.duality.h
  duality.v <- duality.v + other.duality.v

  # Noise does NOT accumulate: the strongest source wins each component
  # independently, because summed grain saturates to white immediately.
  noise.intensity <- max(noise.intensity, other.noise.intensity)
  noise.grain     <- max(noise.grain,     other.noise.grain)
  noise.fps       <- max(noise.fps,       other.noise.fps)

  color_base <- color_base + other.color_base      # per channel
  color_gray <- color_gray + other.color_gray
  color_add  <- color_add  + other.color_add

  # Colour mapping is a two-slot crossfade, filled first-come.
  IF other names a first texture
    IF this block already names one
      cm_tex2        <- other.cm_tex1
      cm_interpolate <- 1 - cm_influence / (cm_influence + other.cm_influence)
    ELSE
      cm_tex1        <- other.cm_tex1
      cm_influence   <- other.cm_influence
      cm_interpolate <- 0
    cm_influence <- max(cm_influence, other.cm_influence)
```

**Invariants** — at most two colour-mapping textures survive a fold. A third contributing effect overwrites the second slot and recomputes the crossfade against the *accumulated* influence, so the blend drifts toward whichever texture arrived first with a large influence. That is a hard limit of the renderer's sampler count, not a modelling choice, and a rebuild with more samplers could keep more.

The crossfade position is an influence *ratio*, not a time: with influences `a` (already accumulated) and `b` (arriving), it lands at `b / (a + b)`. The arithmetic is written as `1 - a/(a+b)`, which is the same number.

**Notes** — the final line raises `cm_influence` to the maximum of the two *after* the ratio has been computed from the old value. Swapping those two statements changes every crossfade, so their order is load-bearing even though it reads like a tidy-up.

Summing `color_base` is arguably wrong — adding two pivots of 0.5 gives 1.0, a pivot at white — but it is what the shipped effects are tuned against, and they compensate by contributing *deltas* from neutral rather than absolute pivots. A rebuild must keep the sum.

## `sub` — remove an effect

**Contract** — subtracts another block's scalars and colours from this one. Deliberately **does not** touch noise or colour mapping, because neither combined additively in the first place and there is nothing to undo.

**Notes** — this asymmetry means fold-then-unfold is not the identity: the noise level and the texture slots stay where the strongest contributor left them. Effects are expected to be rebuilt from scratch each frame rather than incrementally removed, and `sub` exists for the few callers that do the latter. A rebuild that recomputes the accumulator per frame can drop this entirely.

## `lerp` — blend toward a target

**Contract** — blends between two blocks by a factor clamped to `[0, 1]` and **adds** the result into this block rather than replacing it. Asserts that the factor is a finite number before clamping.

**Notes** — the add-rather-replace behaviour is the surprising part and is what makes this composable with `add` at a call site that folds several animated effects in sequence.

Three fields ignore the factor entirely and are taken verbatim from the target: the three noise components. Interpolating grain rate produces visible stepping as the rate sweeps, so it snaps instead. The two colour-mapping texture names likewise snap, while the influence and crossfade position blend — which is what makes a colour-mapping effect fade in smoothly while its textures change instantly.

## `validate`

**Contract** — asserts that every scalar and every colour channel in the block is a finite number, tagging the failure with a caller-supplied label. Compiled out of a shipping build.

**Notes** — this exists because a single non-finite value here propagates into the renderer's constant buffer and turns the whole screen black or white with no other symptom. The label is how a failure is traced back to which effect produced it — worth keeping in any rebuild, in whatever form its checked builds take.

## `normalize`

**Contract** — does nothing.

**Notes** — an empty function retained because callers still invoke it. A rebuild should delete it and the call sites; there is no lost behaviour, only a name that once meant something.
