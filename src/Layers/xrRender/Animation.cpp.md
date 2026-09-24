# src/Layers/xrRender/Animation.cpp

> The four-channel mixing rule table: channels 0 and 1 replace, channels 2 and 3 layer on top.

**Needs** — [`Animation.h`](Animation.h.md)
**Used by** — reached through its declarations in [`Animation.h`](Animation.h.md); callers name that, not this file.
**Tier floor** — T3: a constant table and four weights.

## Purpose

Holds the one decision this whole pair of files exists to record: what each of the four animation channels *means*. Everything else in the pair is accessors.

## State

```text
RECORD channels                    # one per animated model
  factors : list<real> of length 4, all 1.0 at reset

# The rule table, shared by every model, indexed by channel:
#   channel 0 : (intern lerp, extern lerp)
#   channel 1 : (intern lerp, extern lerp)
#   channel 2 : (intern lerp, extern add)
#   channel 3 : (intern lerp, extern add)
```

**Invariants**

- `intern_` is `lerp` for all four channels. Two animations in the same lane are always *blended between*, never summed — summing them would double any motion they share. There is no channel where within-lane addition is correct, which is why the field never varies; it exists so that the record's shape matches the extern side.
- `extern_` is what distinguishes the lanes. Channels 0 and 1 produce a pose that *is* the bone's pose, so the finished channel result is interpolated into what came before. Channels 2 and 3 produce a *delta* pose — a lean, a recoil, a head turn — and are applied as a rotation-and-translation addition on top. This is the reason the system has channels at all.
- Weights reset to 1.0, not 0: a channel that nobody has tuned contributes at full strength. A rebuild that defaults them to zero silently mutes every additive animation in the shipped data.
- A channel index at or past the ceiling is an authoring error and is asserted, not clamped.

## `channels.init` / `channels.set_factor`

**Contract** — reset all four weights to full, or set one of them. No allocation, no device contact. The weight is read once per bone per frame by the mixer, so it is expected to be set from the game layer between frames, not during the solve.

## `channels.rule` / `channels.get_def`

**Contract** — read the rule for a channel, or bundle the rule and the current weight into the record the per-bone mixer consumes. Pure reads.

**Notes** — The rule lookup is a plain table index into shared constant data; the mixer asks per bone and per channel, which means it happens a few hundred thousand times a frame on a crowded level. A rebuild should keep the rule table small enough to sit in cache next to the weights and resist the temptation to make it a virtual dispatch per channel.
