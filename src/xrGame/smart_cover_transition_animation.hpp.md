# src/xrGame/smart_cover_transition_animation.hpp

> Declares one animation entry of a smart-cover transition: where the creature stands, what clip plays, and the posture and gait it implies.

**Needs** — [`ai_monster_space.h`](ai_monster_space.h.md) · [`smart_cover_transition_animation.cpp`](smart_cover_transition_animation.cpp.md) · [`smart_cover_transition_animation_inline.hpp`](smart_cover_transition_animation_inline.hpp.md)
**Used by** — [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`smart_cover_transition.cpp`](smart_cover_transition.cpp.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_transition_animation.cpp`](smart_cover_transition_animation.cpp.md) · [`smart_cover_transition_animation_inline.hpp`](smart_cover_transition_animation_inline.hpp.md) · [`stalker_movement_manager_smart_cover.cpp`](stalker_movement_manager_smart_cover.cpp.md) · [`stalker_movement_manager_smart_cover_loopholes.cpp`](stalker_movement_manager_smart_cover_loopholes.cpp.md)
**Tier floor** — T3: an immutable four-field value read from configuration

## Purpose

Declares the surface implemented in
[`smart_cover_transition_animation.cpp`](smart_cover_transition_animation.cpp.md) — which
is only the constructor; every reader is an accessor defined inline. The type is an
immutable record, and a rebuild should make it exactly that rather than a class.

Exported units: the constructor, plus reads of `position`, `animation_id`, `body_state`,
`movement_type`, and `has_animation` (whether a clip name was authored at all).

## State

```text
RECORD animation_action
  position      : vector         # offset relative to the cover object's transform
  animation_id  : text           # motion name; empty means "no clip, just the posture"
  body_state    : enum posture   # what the creature is in when the entry completes
  movement_type : enum gait      # how it moves while the entry runs
```
