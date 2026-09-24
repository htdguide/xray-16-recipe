# src/xrGame/stalker_danger_property_evaluators.cpp

> Classifying a threat, and the one evaluator that does real work: deciding whether a chosen cover point is still the right one.

**Needs** — [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`danger_object.h`](danger_object.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_evaluators.h`](cover_evaluators.h.md) · [`cover_point.h`](cover_point.h.md) · [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md)
**Tier floor** — T2: one of these searches the level's cover database every planning cycle.

## Purpose

Eight questions. Six are one-line classifications of the selected danger, present so that
the danger planner's preconditions can be written over them; one is about the enemy; and one
— the cover evaluator — is a complete algorithm that happens to be phrased as a question.
That last one is the reason this file is worth reading.

## State

Only the cover evaluator holds any:

```text
RECORD DangerUnknownCoverActual
  cover_selection_position : Position   # where the creature stood when it last chose cover
```

**Invariants** — the remembered position is the *anchor of the current choice*, not the
creature's current position. It is reset to the creature's position whenever the choice is
abandoned, and the difference between the two is what the evaluator's loop is about.

## The danger classification

**Contract** — each of these reads the danger memory's currently selected threat and
answers a boolean about its type. All are pure, all return false when nothing is selected,
none allocates.

```text
Dangers            := a danger is selected

DangerUnknown      := selected.type IN { bullet ricochet,
                                         an entity died nearby,
                                         a fresh corpse was found }

DangerInDirection  := selected.type IN { the sound of someone attacking,
                                         an ally was attacked,
                                         I was attacked,
                                         the sound of an enemy }

DangerGrenade      := selected.type = grenade

DangerBySound      := false                 # see Notes
```

**Invariants** — the four classes must partition the danger types the memory can select, or
the danger planner has no available branch and the creature stands still with a threat it
has classified as nothing. A rebuild that adds a danger type must also assign it to a class.

**Notes** — the split between *unknown* and *in-direction* is the load-bearing distinction
in the whole danger system, and it is a distinction about **information**, not severity. A
ricochet, a body dropping, a corpse: these tell a creature that something is wrong and
nothing about where. A shot fired at you, a comrade being hit, a hostile voice: these carry
a bearing. The two reactions differ accordingly — one sweeps and searches, the other takes
cover facing a direction and looks out.

The sound class is **switched off**: its evaluator checks that a danger is selected and then
returns false regardless. The condition it would use — the enemy-sound type — is instead
claimed by the in-direction class, where the original moved it. The result is that the
by-sound sub-planner exists, is wired up, and is unreachable. A rebuild should either delete
that whole branch or restore the type to it; keeping the dead wiring is the worst of the
three, and it is what the original does.

Three further danger types are commented out of the in-direction class with the note that
they are "fakes, temporarily". They are exactly the three that the unknown class claims, so
the comment records an experiment in treating directionless dangers as directional. It was
not kept, and there is nothing to recover from it.

## `DangerGrenadeExploded` and `GrenadeToExplode`

**Contract** — the two halves of the grenade timeline, distinguished by whether the danger
record still points at a live grenade object.

```text
GrenadeToExplode      := a grenade danger is selected AND its grenade object still exists
DangerGrenadeExploded := a grenade danger is selected AND its grenade object is gone
```

**Invariants** — the danger record outlives the grenade. That is the mechanism: the record
is what lets a creature keep reacting for a moment after the blast — staying down, then
looking around — instead of standing up the instant the object is destroyed.

**Notes** — `GrenadeToExplode` has one extra guard: it answers false while the creature is
playing a whole-body animation. A creature mid-critical-hit or mid-smart-cover-transition
cannot dive for cover, and letting the planner believe it could would abandon the animation
half-played. This is the general rule that whole-body animations are uninterruptible,
enforced at the one place where it would otherwise be violated.

## `EnemyWounded`

**Contract** — is the currently selected enemy a downed creature rather than a fighting
one. Answers false when there is no enemy, and false when the enemy is not the kind of
creature that *can* be wounded — only human-shaped creatures have the downed state; a mutant
is either fighting or dead.

**Invariants** — the wounded test is asked *relative to this creature's movement
restrictions*. A downed enemy that this creature cannot legally reach does not count as
wounded, because the only thing that answer is used for is deciding whether to walk over and
finish it. A rebuild that asks the global question will send creatures at bodies they can
never reach.

## `DangerUnknownCoverActual`

**Contract** — answers "is the cover point I am heading to still the best available one".
Searches the level's cover database, may reserve a point with the squad coordinator, and may
write another proposition as a side effect. Not pure, not cheap, and run on every planning
cycle of the unknown-danger branch.

The question exists because cover selection has to be *stable*. Recomputing the best cover
every cycle and walking toward whatever comes back produces a creature that changes its mind
every few metres and never arrives. So the evaluator's real job is to decide when a
re-selection is justified, and the answer it returns is "no re-selection needed".

```text
FUNCTION evaluate() -> bool
  IF no danger selected THEN RETURN false

  # Re-anchor the search when the previous choice is void, or when the creature
  # walked to the end of a path without having reached cover.
  IF I hold no cover claim THEN anchor := my position
  IF NOT property(CoverReached) AND movement.path_completed THEN anchor := my position
  IF the cover evaluator holds a selection but I hold no claim THEN invalidate it

  previous := my claimed cover point
  threat   := selected danger position
  first_pass := true
  LOOP
    configure the cover evaluator against threat
    point := cover_database.best_cover(near = anchor, radius = 10, evaluator, my restrictions)
    IF point is none THEN
      point := cover_database.best_cover(near = anchor, radius = 30, evaluator, my restrictions)

    IF NOT first_pass THEN BREAK              # second pass takes whatever it found

    IF point is previous THEN result := true; BREAK
    IF previous exists AND point exists
       AND distance(point, previous) <= 1 THEN
      point := previous; result := true; BREAK  # near enough: keep the old claim

    IF anchor is already my position THEN BREAK  # nothing left to try

    anchor := my position                        # retry the search from where I stand
    result := false
    first_pass := false

  squad.reserve_cover(me, point)
  IF NOT result THEN set property CoverReached = false
  RETURN result
```

**Invariants**

- The loop runs at most twice. The first pass searches from the anchor the previous choice
  was made at; if that produces a different point, the second pass searches from where the
  creature actually is now. That is the entire reason the anchor is remembered: searching
  from the old anchor is what makes the answer *stable* as the creature walks, and searching
  from the current position is what makes it *correct* once the old anchor is stale.
- A new point within one unit of the previous one is treated as the previous one. Cover
  points are dense; without this the creature would re-path constantly between
  indistinguishable positions.
- Returning false must also clear `CoverReached`, because the creature is no longer at the
  cover it is being sent to. Writing another planner proposition from inside an evaluator is
  a layering violation and it is deliberate: the two facts are derived from the same search
  and computing them separately would double its cost.

**Notes** — the search is tried at a ten-unit radius and then at thirty. The near radius is
what the behaviour wants (cover close enough to reach before whatever it is arrives); the
wide radius is the fallback that keeps a creature in open ground from having no plan at all.
Preferring near cover and accepting far cover is the difference between a stalker that ducks
behind the nearest crate and one that sprints across a field.

The cover evaluator is configured with the threat's position, a ten-unit minimum standoff,
a hundred-and-seventy-degree arc and a ten-unit spread. Those numbers describe "cover that
faces the threat, from a position not right on top of it". The arc being just short of a
half-turn means a point is acceptable if the threat is anywhere in front of it — cover is
about interposing geometry, not about a firing angle.

Reserving the chosen point with the squad coordinator on every evaluation, including when
nothing changed, is what keeps the claim alive: claims are refreshed by use, so an
evaluator that stopped reserving would have its point taken by an ally on the next cycle.
