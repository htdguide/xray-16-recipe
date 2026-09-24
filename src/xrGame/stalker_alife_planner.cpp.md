# src/xrGame/stalker_alife_planner.cpp

> The stalker's default brain: three world properties and three operators covering the only three things a stalker does when nothing is demanding its attention.

**Needs** — [`stalker_alife_planner.h`](stalker_alife_planner.h.md) · [`stalker_alife_actions.h`](stalker_alife_actions.h.md) · [`stalker_alife_task_actions.h`](stalker_alife_task_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`action_planner_action_script.h`](action_planner_action_script.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a table of preconditions and effects

## Purpose

A planner built as an operator, so that it can be installed inside a larger plan: the
stalker's top-level brain selects "do whatever you normally do" and this sub-plan decides
what that means. It is also script-extensible, which is how the shipped games attach their
own behaviour without touching the engine — the registration is in the base type it derives
from.

The file is almost entirely a declaration of a planning problem. Its content is which
properties exist, which operator changes which, and what each demands — and that table is
the entire behaviour.

## State

`Stateless.` — the planner's own state (current plan, world state, installed operators) lives
in the planner base; this file only fills it.

## The planning problem

```text
Properties
  alife_running      — is the alife simulation active on this level
  smart_terrain_task — does this stalker have an unclaimed job from a smart terrain
  puzzle_solved      — the terminal property; see below

Operators
  free_no_alife            requires: NOT alife_running, NOT puzzle_solved
                           achieves: puzzle_solved
  smart_terrain_task       requires: alife_running, smart_terrain_task
                           achieves: NOT smart_terrain_task
  solve_zone_puzzle        requires: alife_running, NOT smart_terrain_task, NOT puzzle_solved
                           achieves: puzzle_solved
```

**Invariants** — the property named `puzzle_solved` is evaluated by a **constant false**.
It can never actually become true, so the operator that claims to achieve it never completes
and the plan never terminates. That is the point: this is the idle brain, and an idle brain
must keep running. The name is vestigial — it once meant a specific scripted puzzle — and a
rebuild should name it for what it does, something like *nothing_left_to_do*, while keeping
the never-satisfiable evaluator.

The consequence of that shape is a clean precedence order. With the alife simulation
running, a pending smart-terrain job wins because its precondition is exclusive with the
idle operator's; with no job pending, the idle behaviour runs forever; with no alife
simulation at all, the no-alife idle runs forever. Exactly one operator is applicable at any
moment, so the "search" is a lookup and the planner's cost never matters here.

## `setup`

**Contract** — bind the planner to its stalker and to the shared world-state storage, then
rebuild the problem from scratch: clear every installed operator and evaluator, add the
three evaluators, add the three operators. Called when the stalker is reinitialized —
spawn, or a transition back online — not per cycle.

```text
FUNCTION setup(stalker, storage)
  base.setup(stalker, storage)
  clear()                    # drop every operator and evaluator, destroying them
  add_evaluators()
  add_actions()
```

**Invariants** — the clear is unconditional and must come first. `setup` runs again on every
reinitialization, and without it a stalker that goes offline and comes back would accumulate
a second copy of every operator. The planner owns its operators and evaluators, so clearing
destroys them.

## `add_evaluators`

**Contract** — install the three property evaluators. One is a constant, one asks whether
the stalker is currently registered with a smart terrain, one asks whether the alife
simulation exists. Each is created here and owned by the planner from that moment.

## `add_actions`

**Contract** — install the three operators with their preconditions and effects, each under
a fixed identifier from the shared decision vocabulary. Ownership transfers to the planner.

**Notes** — the identifiers come from a project-wide enumeration shared with every other
stalker planner, which is what lets a script replace one operator by identifier without
knowing which planner installed it. That shared namespace is part of the frozen script
surface.
