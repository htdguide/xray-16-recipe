# src/xrAICore/Navigation/graph_engine_space.h

> Fixes the scalar types every search in the engine is measured in, and names the planner's world-state vocabulary.

**Needs** — [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`../Components/operator_condition.h`](../Components/operator_condition.h.md) · [`../Components/condition_state.h`](../Components/condition_state.h.md) · [`../Components/operator_abstract.h`](../Components/operator_abstract.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`problem_solver_inline.h`](../Components/problem_solver_inline.h.md) · [`script_world_property_script.cpp`](../Components/script_world_property_script.cpp.md) · [`script_world_state_script.cpp`](../Components/script_world_state_script.cpp.md) · [`path_manager_solver_inline.h`](PathManagers/path_manager_solver_inline.h.md) · [`graph_engine.h`](graph_engine.h.md) · [`vertex_manager_hash_fixed_inline.h`](vertex_manager_hash_fixed_inline.h.md) · [`member_order.h`](../../xrGame/member_order.h.md) · [`movement_manager.h`](../../xrGame/movement_manager.h.md) · [`object_handler.h`](../../xrGame/object_handler.h.md) · [`property_evaluator.h`](../../xrGame/property_evaluator.h.md) · [`property_storage.h`](../../xrGame/property_storage.h.md) · [`stalker_animation_manager.h`](../../xrGame/stalker_animation_manager.h.md)
**Tier floor** — T2: a naming and width decision. The widths matter but nothing here is device- or format-facing.

## Purpose

One place that decides what a cost *is*, what a vertex identity *is*, and what an iteration count
*is*, separately for navigation and for the planner. Everything downstream — the search, the path
managers, the parameter records — is written against these names, so changing a width is a change
in one file rather than in fifty.

It also names the planner's three core types, which are defined in
[`../Components`](../Components/README.md) but referred to from navigation code.

## State

Stateless. It fixes types and names; the records it names are defined in [`../Components`](../Components/README.md) and [`PathManagers`](PathManagers/README.md).

## The two measurement systems

```text
# navigation: through a level's mesh and across the cross-level graph
distance   : real (32-bit)   # metres of path length
vertex id  : int (32-bit)    # index into a graph's vertex array
iteration  : int (32-bit)    # search step counter

# planner: through the space of world states
distance   : int (16-bit)    # operator cost; small, integral, and summed over a short plan
vertex id  : WorldState      # a set of (condition, value) pairs — not an index
edge       : int (32-bit)    # an operator identifier
condition  : int (32-bit)    # a world-property identifier
value      : bool            # every world property is a predicate
iteration  : int (32-bit)    # shared with navigation
```

**Invariants** — the planner's cost is a 16-bit integer, and that is load-bearing in two ways.
It bounds what an operator may cost and how long a plan may be before the sum saturates, and it
makes the planner's frontier orderable with integer comparisons. A rebuild that widens it to a
real must re-examine the planner's tie-breaking, which currently relies on exact equality of
integer costs.

Every world property is a boolean. The planner cannot express "health below 30%" as a value; the
game must precompute such a thing into a named predicate and publish it as true or false. That
restriction is what keeps the planner's state space enumerable and its state comparison cheap,
and it shapes how every creature's goals are authored.

## `WorldProperty` / `WorldState` / `WorldOperator`

**Contract** — a *world property* is one (condition identifier, boolean value) pair. A *world
state* is a set of them — a partial description of the world, in which a property not mentioned
is simply unconstrained. A *world operator* is an action with a precondition state, an effect
state, and a cost. These are the vertices and edges the planner's search runs over; their
behaviour is in [`../Components`](../Components/README.md).

```text
RECORD SolverConditionValue
  condition : int (32-bit)
  value     : bool
  # equality is by condition alone: a state holds at most one entry per condition
```

**Notes** — comparing a condition-value pair by its condition only, ignoring the value, is how
lookup into a state works: "does this state say anything about condition X". A rebuild must not
"fix" this into full-tuple equality, because the planner relies on it to find and overwrite a
property.

## Parameter record names

Six named parameter bundles are declared here and defined in
[`PathManagers`](PathManagers/README.md): the base bundle (the three search budgets), the
flooder, the straight-line walk, the nearest-vertex search, the game-and-level search, the
game-vertex search, and the planner's own. Naming them here rather than where they are defined
keeps the engine's search entry points writable without pulling in every path manager.

## `CScriptWorldProperty` / `CScriptWorldState`

**Contract** — the two planner types that are exported to the script layer, so that a mod can
construct a goal state and hand it to a creature's planner. They carry only a registration hook;
the exported surface itself is declared where the registration runs.
