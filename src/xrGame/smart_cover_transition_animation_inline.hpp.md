# src/xrGame/smart_cover_transition_animation_inline.hpp

> Field reads on a smart-cover transition animation entry, and the one rule that an empty motion name means "posture change only".

**Needs** — [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md)
**Used by** — [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md)
**Tier floor** — T4: field reads

## Purpose

Separated from the declaration for C++ compilation reasons only; fold these into the
record in a rebuild. One line here is not a plain accessor and carries a decision.

## State

`Stateless.`

## `has_animation`

**Contract** — reports whether this entry names a motion at all, by testing the motion
name against the empty string. An entry with no motion is legitimate authored data: it
means the transition changes the creature's posture and position without playing a clip.
Callers must check this before asking the animation system to play the name, because an
empty name is not a valid motion lookup.

## `position`, `animation_id`, `body_state`, `movement_type`

**Contract** — return the corresponding authored field. No side effects.
