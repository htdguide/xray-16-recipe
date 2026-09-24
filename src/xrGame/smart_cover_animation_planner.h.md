# src/xrGame/smart_cover_animation_planner.h

> Declares the planner that runs a creature's whole life inside a smart cover: the sub-plan that alternates idling, looking out, firing, reloading and leaving.

**Needs** — [`action_planner_script.h`](action_planner_script.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md)
**Used by** — [`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) · [`smart_cover_animation_planner_inline.h`](smart_cover_animation_planner_inline.h.md) · [`smart_cover_animation_selector.cpp`](smart_cover_animation_selector.cpp.md) · [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) · [`smart_cover_default_behaviour_planner.cpp`](smart_cover_default_behaviour_planner.cpp.md) · [`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md) · [`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) · [`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md) · [`smart_cover_planner_target_selector.cpp`](smart_cover_planner_target_selector.cpp.md)
**Tier floor** — T2: a nested planner with per-creature timing state

## Purpose

Declares the surface implemented in
[`smart_cover_animation_planner.cpp`](smart_cover_animation_planner.cpp.md) and
[`smart_cover_animation_planner_inline.h`](smart_cover_animation_planner_inline.h.md).

This is a planner that is itself an *action* of the creature's outer planner. Once a
creature has decided to use a cover, this takes over and plans within it. Being a planner
and an action at once is what makes smart-cover behaviour composable: the outer plan says
"be in cover", and everything that happens there — the peeking rhythm, the choice of
loophole, the decision to reload — is a self-contained search this object runs.

It is also the owner of the *timing* state that the dwell evaluators read and write, and
of the per-creature random stream that staggers a squad.

## Exported units

- **The class** — a script-extensible planner over the creature, with its own world state.
- **`setup`** — attach to a creature, build the evaluators and operators, and hand the
  planner to the movement manager's target selector.
- **`initialize` / `finalize`** — the in-cover lifecycle: hook the hit callback, save and
  later restore the creature's head turn speed, seed the world state, and choose a default
  goal.
- **`update`** — one planning and execution cycle.
- **`target`** — set the goal as a single world property that must become true.
- **`time_object_hit`** — when this creature was last hit; read by a dwell evaluator.
- **`loophole_value` / `decrease_loophole_value`** — a decaying preference score for the
  current loophole.
- **`last_transition_time`** — when the last animated transition was made.
- **`default_idle_interval` / `default_lookout_interval`** — a *fresh random draw* each
  time, between the authored bounds.
- **`idle_min_time` / `idle_max_time` / `lookout_min_time` / `lookout_max_time`** — those
  bounds, in seconds, set by the loophole actions.
- **`stay_idle`, `last_idle_time`, `last_lookout_time`** — the phase flag and the two
  phase start stamps the dwell evaluators share.
- **`property_storage`** — the planner's world state, handed to nested machinery.
- **`object_name` / `cName`** — identity for diagnostics.

## State

```text
RECORD animation_planner
  target                   : world state          # the goal, always a single property
  time_object_hit          : int (ms)
  loophole_value           : int, starts at 1000  # decaying preference; see the notes
  last_transition_time     : int (ms)
  head_speed               : real                 # the creature's own, saved on entry
  random                   : random stream        # per planner instance
  idle_min_time, idle_max_time         : real (seconds)
  lookout_min_time, lookout_max_time   : real (seconds)
  stay_idle                : bool, starts true
  last_idle_time, last_lookout_time    : int (ms)
```

**Invariants** — the four time bounds are in *seconds* while every stamp and interval is in
*milliseconds*; the conversion happens in the interval draws. Exactly one of the idle and
lookout phases is live, named by `stay_idle`.

## Notes

The loophole value starts at 1000 and only ever decreases. Nothing in this file or its
implementation reads it for a decision — the evaluator constructed with it ignores it — so
it is a scoring mechanism that was built and then left unwired. A rebuild should leave it
out; recording it here is the honest answer to "what was this for", which is: a decaying
preference intended to make a creature eventually abandon a loophole it has been using,
never connected to the search.
