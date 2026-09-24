# src/xrGame/smart_cover_planner_actions.h

> Declares the base every smart-cover action shares, and the three operators that move a creature between loopholes and out of a cover.

**Needs** — [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) · [`smart_cover_loophole_planner_actions.h`](smart_cover_loophole_planner_actions.h.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`smart_cover_planner_actions_inline.h`](smart_cover_planner_actions_inline.h.md)
**Tier floor** — T2: planner operators driving the animation layer

## Purpose

Declares the surface implemented in
[`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md).

The base class here is the important part: it is what the animation selector
([`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md)) requires of
*any* smart-cover operator. A rebuild reading only this file learns the contract every
in-cover action must satisfy.

## `action_base`

**Contract** — an abstract combat action extended with the four things the animation
selector asks of it. It is an interface, so what it demands of an implementor is the
contract a rebuild must satisfy:

- **`select_animation`** — required. Name the clip to play next. Only called when the
  action reports itself animated.
- **`on_animation_end`** — required. The clip finished; commit whatever the action was
  going to change. This, not `execute`, is where a smart-cover action's *effect* happens.
- **`on_mark`** — optional (default: nothing). The clip's event marker was crossed.
- **`on_no_mark`** — optional (default: nothing). The marker was not crossed this query;
  gives an action a per-query hook.
- **`is_animated_action`** — optional (default: yes). Whether this action plays a
  smart-cover clip at all. A "no" hands the creature back to its ordinary animation.
- **`setup_orientation`** — a helper, not a hook: re-enable the creature's aiming and
  reinstate its bone callbacks. Every action that hands the creature back to normal
  control calls it.

**Invariants** — the marker hooks are called on **every** query, one or the other, never
neither and never both. An action may therefore use the pair as a per-frame tick, and one
of them does.

## `change_loophole`

**Contract** — the animated move: play the transition's clip and, when it ends, commit the
creature to the next loophole. Used for both moving within a cover and leaving it with an
animation.

## `non_animated_change_loophole`

**Contract** — the unanimated move: no clip, the creature walks. Reports itself not
animated.

## `exit`

**Contract** — leaving the cover, deciding *at run time* whether it is animated by asking
whether the pending transition has a clip. The only action whose animated-ness is dynamic.

## Notes

The inline sibling
([`smart_cover_planner_actions_inline.h`](smart_cover_planner_actions_inline.h.md)) is
empty. It exists so the header can be included uniformly with the rest of the smart-cover
headers; a rebuild should delete it.
