# src/xrGame/stalker_death_planner.cpp

> Two operators: die, then be dead. The second does nothing, and that is its purpose.

**Needs** — [`stalker_death_planner.h`](stalker_death_planner.h.md) · [`stalker_death_actions.h`](stalker_death_actions.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_death_planner.h`](stalker_death_planner.h.md)
**Tier floor** — T2: a two-operator search that runs once and then idles forever.

## Purpose

The branch the root planner selects when the creature is not alive. It exists because a
dead creature must still be *running something*: the brain's cycle continues while the body
ragdolls, the spasm plays and the inventory settles, and the planner needs a legal plan the
whole time or it will log a failure every cycle for the rest of the level.

## State

`Stateless.`

## `setup`

**Contract** — bind, clear the `Dead` proposition, and rebuild both tables. Clearing on
setup matters for the same reason as everywhere else: setup also runs when a save is
loaded, and a creature loaded from a save in which it was alive must not start its brain
believing it has already finished dying.

## `add_evaluators`

```text
PuzzleSolved : constant false     # the author's name for it: "resurrecting"
Dead         : latched by the dying action
```

**Notes** — `PuzzleSolved` is the root planner's never-satisfiable goal
(see [`stalker_planner.cpp`](stalker_planner.cpp.md)), and inside this branch it is pinned
to false. That pin is what makes the branch terminal: the goal can never be observed as
already met, so the plan is always available and the planner always has something to run.
Naming the evaluator "resurrecting" is a joke about what it would mean for the answer to
ever be true.

## `add_actions`

```text
Dying   requires  Dead = false
        effects   Dead = true

Dead    requires  Dead = true
        effects   PuzzleSolved = true
```

**Invariants** — the second operator is an instance of the *plain* action base, with no
behaviour of its own: the base's entry clears any script animation queue and its per-cycle
hook does nothing. It is a placeholder whose only job is to be a valid, permanently
selectable action so that the brain has something to sit on.

**Notes** — the two-step shape is the general solution to "this entity is finished but its
object still exists". The first operator does the work, once; the second absorbs every
subsequent cycle. A rebuild that instead unregisters the creature from the scheduler at
death will be faster, but must then handle the cases the original leaves to the idle
action: a corpse that is still settling, still being looted, still holding a weapon it
could not drop, and — in the alife simulation — still a record that may be promoted back
online.
