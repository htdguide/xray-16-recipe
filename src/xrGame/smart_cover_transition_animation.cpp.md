# src/xrGame/smart_cover_transition_animation.cpp

> Constructs the immutable animation entry of a smart-cover transition.

**Needs** — [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md)
**Used by** — [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md)
**Tier floor** — T3: field initialisation only

## Purpose

The whole file is one constructor that stores four authored values. It is a separate
translation unit for compilation reasons that vanish in a rebuild; merge it into the
record's declaration.

## State

`Stateless.` — the record it builds is described in
[`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md).

## `animation_action`

**Contract** — takes a position offset, a motion name, an end posture and a gait, and
stores all four. No validation, no allocation, no failure mode. The values are never
modified afterwards, which is the load-bearing part: transition animations are shared
between every creature using the cover, so they must be read-only once built.
