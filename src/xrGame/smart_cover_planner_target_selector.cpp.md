# src/xrGame/smart_cover_planner_target_selector.cpp

> The top of the smart-cover stack: decides whether a creature in a cover should look out, fire, fire blind, or fall back to peaceful behaviour, and tells the script layer every cycle.

**Needs** — [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md) · [`smart_cover_planner_target_provider.h`](smart_cover_planner_target_provider.h.md) · [`smart_cover_default_behaviour_planner.hpp`](smart_cover_default_behaviour_planner.hpp.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a five-operator plan search per cycle, plus a script call

## Purpose

Five operators that answer one question: *what should this creature in this cover be doing
right now?* Every cycle the answer is handed down to the animation planner as a goal, which
turns it into clips.

The selector is also the place the script layer is told what is happening, once per cycle,
unconditionally.

## State

Declared in
[`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md): a
script callback and a random stream.

## `setup`

**Contract** — registers the evaluators and operators, sets the goal, and **seeds the
"looked out" flag randomly**: true with probability 0.7, false otherwise.

```text
FUNCTION setup(animation_planner, world_state)
  world state[looked out]            = uniform draw <= 0.7
  world state[too much time firing]  = false
  register evaluators and operators
  goal = has target
```

**Invariants** — the random seeding is the single most consequential line in the file. The
fire operator requires having looked out; the lookout operator requires not having. Seeding
the flag randomly means that of a squad entering cover together, about seven in ten open
fire immediately and the rest peek first. Seeding it to a constant makes a squad act in
unison, which is exactly what smart covers exist to avoid. The bias toward true is because
a creature that has just chosen to take a combat cover is usually reacting to something it
has already seen.

## `add_evaluators`

**Contract** — registers nine questions:

- **looked out** and **too much time firing** — read back out of the planner's own world
  state; these carry across cycles.
- **last hit was long ago** — has it been at least sixteen seconds since the creature was
  hit here.
- **can look out / can fire / can fire without looking out** — does the current loophole
  authorize each of the three actions.
- **use default behaviour** — is the creature in default or combat cover behaviour.
- **can fire at enemy** — may it shoot from where it stands.
- **has target** — pinned false, the goal.

**Invariants** — the sixteen-second hit window is the longest timing constant in the smart
cover system and it drives the blind-fire behaviour below. It is not configurable.

## `add_actions` — the operator table

**Contract** — registers five operators.

```text
idle              needs  too_much_time_firing
                  gives  NOT too_much_time_firing
                  inertia 1000 ms
                  # the enforced pause after a firing spell

lookout           needs  loophole offers "lookout", NOT default_behaviour,
                         NOT looked_out, last_hit_long_ago, NOT has_target
                  gives  has_target;  animation planner goal = looked_out

fire              needs  can_fire_at_enemy, loophole offers "fire", looked_out,
                         last_hit_long_ago, NOT too_much_time_firing, NOT has_target
                  gives  has_target;  animation planner goal = loophole_fire

fire_no_lookout   needs  can_fire_at_enemy, loophole offers "fire_no_lookout",
                         NOT last_hit_long_ago, NOT has_target
                  gives  has_target;  animation planner goal = loophole_fire_no_lookout

default_behaviour needs  default_behaviour, NOT too_much_time_firing, NOT has_target
                  gives  has_target;  runs the nested peaceful planner
```

**Invariants** —

- **Being hit recently is what makes a creature fire blind.** The lookout and fire
  operators both require the last hit to have been long ago; the blind-fire operator
  requires the opposite. A creature under fire stops exposing itself and shoots over the
  wall without looking — and sixteen seconds after the last hit it starts peeking again.
  This one condition, inverted between three operators, is the whole suppression mechanic.
- **Peeking must precede firing.** The fire operator requires "looked out" and the lookout
  operator requires its negation, so the two alternate. Blind fire is exempt, which is why
  it clears the flag itself
  ([`smart_cover_planner_target_provider.cpp`](smart_cover_planner_target_provider.cpp.md)).
- **The idle operator is the only one without the "no target yet" precondition** and the
  only one that does not produce a target. It is a *pure interrupt*: when the firing limit
  trips, it is the only applicable operator, it runs for its one-second inertia, and it
  clears the flag. A rebuild that treats it as a peer of the others will let the planner
  choose it when nothing has tripped.
- **Peaceful and combat behaviour are mutually exclusive by precondition**, on the same
  evaluator. The nested planner never runs alongside the combat operators.

**Notes** — the idle operator carries four commented-out preconditions in the original that
would have made it applicable in the normal course of things rather than only as an
interrupt. They are disabled, and the behaviour they describe — an idle chosen as a peer
option — is not what ships. A rebuild should implement the interrupt.

## `update`

**Contract** — runs one planning cycle, then invokes the script callback with the
creature's [game object](../../GLOSSARY.md) facade. The callback is invoked
**unconditionally**, whether or not one is installed and whether or not the plan changed.

**Notes** — the original marks this with a note wondering whether the planning cycle should
be skipped when a script callback is installed, i.e. whether a script that takes over should
suppress the engine's own decision. It does not, and shipped scripts rely on the engine
continuing to plan underneath them. A rebuild that adds the skip changes modded behaviour.

## `callback` / `object_name`

**Contract** — installs the script callback, replacing any previous one, and supplies a
fixed diagnostic identity.
