# src/xrGame/agent_manager_space.h

> The vocabulary of the squad planner: the four world properties it reasons over and the four operators it plans with.

**Needs** — _(none)_
**Used by** — [`agent_manager_planner.cpp`](agent_manager_planner.cpp.md) · [`agent_manager_properties.cpp`](agent_manager_properties.cpp.md) · [`member_order.h`](member_order.h.md)
**Tier floor** — T3: two enumerations.

## Purpose

The planner in this engine is a search from the current world state to a goal state over
**operators** whose preconditions and effects are stated as **properties**. Both sides
need a shared identifier space. This file is the squad planner's, kept separate from the
planner itself so that evaluators, actions and the planner can all name a property without
depending on each other.

## State

```text
ENUM AgentProperty
  Orders   = 0    # the squad has a current order; this is the goal
  Item     = 1    # some member has selected an item worth collecting
  Enemy    = 2    # some combat member has selected an enemy
  Danger   = 3    # some member has selected a danger
  Script   = 4    # first identifier a script-added property may use
  Dummy    = -1   # "no property"; the maximum value of the width, so it can never
                  #   collide with a real one

ENUM AgentOperator
  NoOrders     = 0
  GatherItem   = 1
  KillEnemy    = 2
  ReactOnDanger = 3
  Script       = 4   # first identifier a script-added operator may use
  Dummy        = -1
```

**Notes** — The reserved `Script` value is the load-bearing part: the script layer adds its
own properties and operators to planners at runtime, and it must not collide with the
engine's. Every planner space in this codebase follows the same convention — engine values
first and contiguous from zero, one named boundary after them, scripts numbering upward
from there. A rebuild that renumbers must keep the boundary.
