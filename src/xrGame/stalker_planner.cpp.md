# src/xrGame/stalker_planner.cpp

> The root of a stalker's brain — the small symbolic world model, and the six
> mutually exclusive modes of being that compete to satisfy it.

**Needs** — [`stalker_planner.h`](stalker_planner.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_alife_planner.h`](stalker_alife_planner.h.md) · [`stalker_anomaly_planner.h`](stalker_anomaly_planner.h.md) · [`stalker_death_planner.h`](stalker_death_planner.h.md) · [`stalker_danger_planner.h`](stalker_danger_planner.h.md) · [`stalker_combat_planner.h`](stalker_combat_planner.h.md) · [`stalker_alife_actions.h`](stalker_alife_actions.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_planner.h`](stalker_planner.h.md)
**Tier floor** — T2: a search over a handful of symbolic operators per creature per cycle.

## Purpose

This is the top of a stalker's decision hierarchy. It holds a world state of seven
booleans, six operators that change them, and one goal. Every other planner in the stalker
is reached *through* one of these operators: five of the six operators are themselves
planners, so the brain is a tree whose root has only seven facts to reason about, and
whose depth is where the detail lives.

The design decision worth carrying into a rebuild is that scale. The root planner is
deliberately tiny. Planning is a search over operators, and a search over six operators
with seven propositions is free; a flat planner with every stalker action in it would not
be. The hierarchy is what makes the planner affordable per creature per frame.

## State

```text
RECORD StalkerPlanner
  affect_cover : bool      # reset to false on every setup; see the inline twin
  # plus the planner base's own evaluator table, operator table, world state and goal
```

## The world model

Seven propositions. Each is answered by an **evaluator** — a small object that inspects the
creature and returns a boolean — re-asked every planning cycle rather than cached:

```text
AlreadyDead   : constant false      # a placeholder the death planner writes over
PuzzleSolved  : constant false      # the goal proposition; see below
Alive         : is the creature alive
Enemy         : is there an enemy, or was there one recently
Danger        : is there a recorded danger
Anomaly       : is there an undetected anomaly nearby
Items         : is there an item worth picking up
```

Two of these are constants, and that is not laziness. `PuzzleSolved` is the goal: the
planner is asked to reach a state where it is true, and *nothing ever makes it true*.
Every operator claims it as an effect, so the planner always has something to do and never
terminates — which is exactly what a creature's brain should do. Naming the
never-satisfiable goal "puzzle solved" is the original's joke; a rebuild should keep the
mechanism and may rename it.

`AlreadyDead` is evaluated as a constant here and is meaningful only inside the death
sub-planner, which supplies its own answer.

The `Enemy` evaluator is the one with a parameter: it reports true for a fixed interval
*after* the last enemy was lost, so that a stalker does not drop out of combat the instant
it loses sight of its target. The interval is the combat planner's own post-combat wait, so
the two agree by construction rather than by two copies of a number.

## The six operators

Each operator is listed with the preconditions it demands of the world state and the
effects it claims. The preconditions are what make the six mutually exclusive, and the
exclusivity is the priority order: read the table as "the highest branch whose conditions
hold wins", because only one set can hold at a time.

```text
DeathPlanner    requires  Alive = false,  PuzzleSolved = false
                effects   PuzzleSolved = true

CombatPlanner   requires  Alive = true,   Anomaly = false,  Enemy = true
                effects   Enemy = false

AnomalyPlanner  requires  Alive = true,   Anomaly = true
                effects   Anomaly = false

DangerPlanner   requires  Alive = true,   Enemy = false,  Anomaly = false,  Danger = true
                effects   Danger = false

GatherItems     requires  Alive = true,   Enemy = false,  Anomaly = false,
                          Danger = false, Items = true
                effects   Items = false

ALifePlanner    requires  Alive = true,   Enemy = false,  Anomaly = false,
                          Danger = false, Items = false,  PuzzleSolved = false
                effects   PuzzleSolved = true
```

Reading the preconditions gives the creature's priorities without any explicit priority
number: death outranks everything; an anomaly outranks even an enemy (walking into a
gravity anomaly kills you faster than a rifle); an enemy outranks a remembered danger;
a danger outranks picking things up; and the off-screen-life planner — going to work,
sleeping, patrolling, following orders from a smart terrain — is what a stalker does when
nothing else is happening. It is the only operator besides death that claims the goal, so
it is the default branch, reached exactly when all five hazards are absent.

`GatherItems` is the only leaf: an ordinary action rather than a sub-planner.

**Invariants** — the sub-planners are created and owned by this planner; installing them
replaces any previous set, so setup must clear before it adds.

## `setup`

**Contract** — binds the planner to a stalker, clears any previous evaluator and operator
tables, installs the seven evaluators and six operators above, sets the goal to
"puzzle solved", and lowers the cover-affecting flag. Idempotent: calling it again rebuilds
the whole table from scratch, which is what happens when a creature is re-initialized after
a save load.

**Invariants** — the goal must be set *after* the operators, since a goal referring to a
proposition with no producing operator makes every plan fail.

## `update`

**Contract** — one planning-and-execution cycle, delegated to the planner base: re-evaluate
the propositions, re-plan if the world state changed or the current plan became invalid,
and execute the head action. The time delta is accepted for signature compatibility with
the scheduler and is not used here — the base drives itself from the same global clock the
evaluators read.

**Notes** — in diagnostic builds this function also toggles planner logging on and off from
a global debug flag on each cycle, so that logging can be switched on mid-game for one
creature at a time, and dumps the entire evaluator, goal and operator tables when planning
fails. A rebuild should keep some form of "why did planning fail" dump: a planner that
silently does nothing is the hardest failure in this codebase to diagnose, because it
looks exactly like a creature deciding to stand still.
