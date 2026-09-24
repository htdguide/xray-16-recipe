# src/xrGame/smart_cover_default_behaviour_planner.cpp

> The peaceful half of cover behaviour: with no enemy to fight, alternate between staying down and looking out, on the dwell timers.

**Needs** — [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) · [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — reached through its declarations in [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md); callers name that, not this file.
**Tier floor** — T2: a two-operator plan search

## Purpose

Two operators and five questions. This is what a creature in a cover with nothing to shoot
at does, and it is the layer that produces the game's most recognizable ambient
behaviour: a guard behind a barricade who periodically bobs up, looks around, and drops
back.

The rhythm is not scripted here. It comes entirely from the two dwell evaluators
([`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md)), which alternate exclusive
readiness and hand over to each other on expiry. This planner just offers the two actions
those readinesses gate.

## State

Declared in
[`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md).

## `setup`

**Contract** — attaches to the animation planner and the outer world state, registers the
evaluators and operators, and sets the goal to "this planner has produced a target".

**Invariants** — the goal is a property whose evaluator is pinned false
([`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md)), so the search never
terminates satisfied and the planner re-chooses an action every cycle. That is exactly the
intent: the point of this planner is to keep producing a new target, not to reach a state.

## `add_evaluators`

**Contract** — registers five questions:

- **has a target** — pinned false, the goal.
- **can stay idle** / **can look out** — does the creature's current loophole authorize
  that action at all.
- **ready to idle** / **ready to look out** — the two dwell timers, each constructed with
  a *fresh random draw* from the animation planner as its first interval.

**Invariants** — the first interval is drawn at registration, which is once per creature,
and every later interval is redrawn on expiry by the evaluator itself. The registration
draw is what staggers creatures relative to each other from the very first cycle; without
it every creature in a squad would peek together on its first pass.

## `add_actions`

**Contract** — registers two operators, each a target provider that names a world property
for the animation planner to pursue.

```text
idle     needs  loophole offers "idle",    ready_to_idle,    NOT has_target
         gives  has_target,  and sets the animation planner's goal to loophole_idle

lookout  needs  loophole offers "lookout", ready_to_lookout, NOT has_target
         gives  has_target,  and sets the animation planner's goal to looked_out
```

**Invariants** —

- The two are **mutually exclusive by construction**, because their readiness
  preconditions are the two halves of one exclusive phase flag. The planner never chooses
  between them; the timers do.
- A loophole that does not authorize an action makes the corresponding operator
  inapplicable, so a firing-slit loophole with no lookout action simply idles forever.
  That is the authored way to make a creature stay down.
- Each operator is set up with the planner's own world state at registration, not lazily.
  A target provider writes into that state when it starts, so it must hold it before the
  first cycle.

## `initialize` / `update` / `finalize` / `object_name`

**Contract** — the three lifecycle calls delegate wholly to the base planner; they exist
because the interface demands them. `object_name` supplies a fixed diagnostic identity.
