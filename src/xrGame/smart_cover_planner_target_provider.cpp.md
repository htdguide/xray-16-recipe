# src/xrGame/smart_cover_planner_target_provider.cpp

> Hands the animation planner its next goal, and decides when a creature has spent long enough firing that it should drop back into cover.

**Needs** — [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`Weapon.h`](Weapon.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: planner glue with one timing rule

## Purpose

Three of these four types are glue. The fourth — the fire provider — contains a real
decision, and it is the one that stops a creature in cover from firing continuously
forever: a timed limit on a firing spell, scaled by how many enemies the squad faces, and
waived when the creature is nearly out of ammunition.

## `target_provider`

**Contract** — on initialization, sets the animation planner's goal to the world property
this provider was constructed with, marks that property true in its own planner's world
state, and subtracts its configured amount from the loophole preference score. Everything
else delegates to the base action.

```text
FUNCTION initialize()
  animation_planner.goal = my world property
  my planner's world state[my world property] = true
  animation_planner.decrease_loophole_value(my loophole value)
```

**Invariants** — marking the property true in the *upper* planner's state is what satisfies
the upper plan's goal, while setting the goal on the *lower* planner is what actually makes
the creature act. Both must happen; doing only the first leaves the creature idle with a
satisfied plan.

**Notes** — every construction site passes zero as the loophole value, so the decay never
happens. See [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) for
what that score was for.

## `target_idle`

**Contract** — once the action's inertia has elapsed, clears the "too much time firing"
flag.

**Invariants** — the clear is gated on completion, not on entry. The flag is what made the
planner choose idle in the first place; clearing it immediately would let the plan flip
straight back to firing and the creature would never actually take cover. The inertia is
the duration of the enforced pause.

## `target_fire`

**Contract** — sets its own inertia at initialization from the squad's enemy count, and on
completion raises the "too much time firing" flag unless the creature is nearly dry.

```text
FUNCTION initialize()
  IF the squad has more than one enemy
    inertia = 6000 ms + a uniform draw up to 3000 ms
  ELSE
    inertia = 0
  then the base initialization

FUNCTION execute()
  IF inertia is zero                 RETURN        # no limit against a lone enemy
  IF not yet completed               RETURN
  IF the creature is ready to kill AND its best weapon's remaining rounds
     are at most a sixth of a magazine             RETURN     # let it finish
  world state[too much time firing] = true
```

**Invariants** —

- **Against a single enemy there is no limit at all.** A creature facing one opponent keeps
  firing until something else stops it. The limit exists to make a *group* fight look like
  a fight rather than a firing line: with several enemies each creature takes turns.
- The interval is six to nine seconds, drawn per firing spell from the **global** random
  stream, so two creatures entering a firing spell in the same frame still get different
  limits.
- The nearly-dry waiver is the interesting case. A creature with at most a sixth of a
  magazine left is allowed to keep firing past its limit, because dropping into cover now
  would mean popping up again in a second to finish the magazine, and then reloading. The
  waiver makes the creature spend the remainder and then reload behind cover, which is both
  what a person would do and what reads correctly on screen.
- The waiver checks the *best weapon to kill with* rather than the active item, so a
  creature that has been commanded to hold a different item is judged on the weapon it
  would actually use.

## `target_fire_no_lookout`

**Contract** — clears the "looked out" flag before the base initialization runs.

**Invariants** — firing blind must not count as having looked out. If it did, a creature
would satisfy its peek obligation without ever exposing itself, and the lookout operator in
the target selector — which requires *not* having looked out — would never become
applicable again.
