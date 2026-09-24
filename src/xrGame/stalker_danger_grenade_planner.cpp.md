# src/xrGame/stalker_danger_grenade_planner.cpp

> The grenade branch: one plan before the blast and another after it, separated by a proposition the world decides.

**Needs** — [`stalker_danger_grenade_planner.h`](stalker_danger_grenade_planner.h.md) · [`stalker_danger_grenade_actions.h`](stalker_danger_grenade_actions.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_grenade_planner.h`](stalker_danger_grenade_planner.h.md)
**Tier floor** — T2: a five-operator search per cycle.

## Purpose

Structurally the unknown-danger branch with one extra proposition threaded through it. That
proposition — has the grenade gone off — is not written by any action; it is read off the
world, and it is what splits the branch into a before and an after without any explicit
sequencing.

## State

`Stateless.`

## `setup` and `initialize`

**Contract** — `setup` binds and rebuilds both tables. `initialize` clears the two latched
progress propositions on entering the branch.

**Notes** — unlike the unknown-danger planner, this one does *not* release the creature's
cover claim at entry. A creature already behind cover when a grenade lands keeps that cover
as the starting point of its escape; the cover evaluator will move it if the grenade makes
the point untenable, and re-deciding from scratch would throw away a head start the
creature cannot afford against a fuse.

## `add_evaluators`

```text
Danger           : is a danger still selected                     (computed)
CoverActual      : is my chosen cover still the best              (computed; the same
                                                                   cover search the
                                                                   unknown-danger branch uses)
CoverReached     : have I arrived                                 (latched by an action)
GrenadeExploded  : has the grenade object ceased to exist         (computed)
LookedAround     : have I finished sweeping                       (latched by an action)
```

**Notes** — the cover evaluator is literally the unknown-danger one, reused. Its search is
anchored on the *selected danger's position*, which for a grenade danger is the grenade,
so the same code picks cover away from a noise and cover away from a grenade without
knowing the difference. That reuse is the reason the evaluator takes its threat position
from the danger record rather than from a parameter.

`GrenadeExploded` being derived from the grenade object's continued existence, rather than
from a blast event, is what makes the branch robust to a grenade that is picked up, removed
by a script, or destroyed some other way: any of those ends the waiting just as a
detonation would.

## `add_actions`

```text
TakeCover                requires  (nothing)
                         effects   CoverActual = true,  CoverReached = true

WaitForExplosion         requires  CoverActual = true, CoverReached = true,
                                   GrenadeExploded = false
                         effects   GrenadeExploded = true

TakeCoverAfterExplosion  requires  GrenadeExploded = true
                         effects   CoverActual = true,  CoverReached = true

LookAround               requires  GrenadeExploded = true, CoverActual = true,
                                   CoverReached = true,    LookedAround = false
                         effects   LookedAround = true

Search                   requires  GrenadeExploded = true, CoverActual = true,
                                   CoverReached = true,    LookedAround = true
                         effects   Danger = false
```

**Invariants** — `WaitForExplosion` claims an effect it cannot cause. The creature does not
make the grenade explode; the planner needs *some* operator that produces
`GrenadeExploded = true` or it can never plan a route to the goal, and waiting is what
produces it in practice. This is the planner idiom for "time passing is an action": a
rebuild that models waiting some other way still needs an operator standing for it, or the
search finds no plan at all and the creature stands in the open.

The two take-cover actions have deliberately asymmetric preconditions: the first has none,
the second requires the explosion. That is what makes the first the universal restart and
the second reachable only after the blast — and what makes the creature move twice.

**Notes** — the branch ends when `Search` runs and the danger record ages out. Compared with
the unknown-danger branch, `Search` here is inert: the creature waits rather than publishing
the spot as dangerous. See
[`stalker_danger_grenade_actions.cpp`](stalker_danger_grenade_actions.cpp.md) for why.

## `update` and `finalize`

**Contract** — pure delegation.
