# src/xrGame/agent_manager_planner.cpp

> Wires the squad's planner: four evaluators that read the pooled squad picture, four operators that act on it, and the standing goal "the squad has an order".

**Needs** — [`agent_manager_planner.h`](agent_manager_planner.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_manager_space.h`](agent_manager_space.h.md) · [`agent_manager_actions.h`](agent_manager_actions.h.md) · [`agent_manager_properties.h`](agent_manager_properties.h.md)
**Used by** — [`agent_manager_planner.h`](agent_manager_planner.h.md)
**Tier floor** — T3: a table of preconditions and effects.

## Purpose

The squad decides what kind of situation it is in — nothing happening, loot on the ground,
a fight, incoming danger — by planning, not by a switch statement. This file supplies the
planner's problem: the properties it may observe, the operators it may apply, and the goal
it searches toward. The search itself is generic and lives elsewhere.

## `setup`

**Contract** — Binds the planner to a squad manager and installs its problem definition.
Clears anything previously installed, registers the four evaluators, registers the four
operators, and sets the target state to the single condition *Orders is true*. Idempotent:
calling it again rebuilds the problem from scratch.

**Invariants** — Because the goal is *Orders is true* and the evaluator for `Orders` is a
constant that always answers *false*, the planner can never reach its goal by observation
alone — it must always select some operator. This is the trick that makes the planner a
situation classifier: the goal is unreachable on purpose, so the plan is always non-empty
and the first step of that plan is the squad's current mode.

## The operator table

```text
# Each row: operator, preconditions, effects.
# The planner picks the cheapest chain from the observed state to Orders=true.

NoOrders       requires Orders=false, Item=false, Danger=false, Enemy=false
               achieves Orders=true
GatherItem     requires Item=true, Enemy=false, Danger=false
               achieves Item=false
KillEnemy      requires Enemy=true
               achieves Enemy=false
ReactOnDanger  requires Enemy=false, Danger=true
               achieves Danger=false
```

**Notes** — The precedence the squad actually exhibits falls out of the preconditions
rather than being written anywhere: an enemy suppresses both looting and danger reaction,
danger suppresses looting, and only a completely quiet world reaches `NoOrders`, the one
operator that satisfies the goal. `KillEnemy` alone has no negative precondition — combat
outranks everything. A rebuild that hard-codes this precedence as an if-chain reproduces
today's behaviour but loses the property the design bought: scripts can add operators and
properties to this space at runtime (see the reserved script boundary in
[`agent_manager_space.h`](agent_manager_space.h.md)) and the precedence re-derives itself.

## The evaluator table

```text
Orders  -> constant false                      # see the invariant above
Item    -> any member has selected an item
Enemy   -> any *combat* member has selected an enemy
Danger  -> any member has selected a danger
```

**Notes** — `Enemy` is polled over the *combat* subset of the roster while the other two
are polled over the whole roster: a member who cannot fight still reports loot and danger
but must not be able to put the squad into combat.

## `remove_links`

**Contract** — Nothing to do. The planner holds no references to game objects; its
evaluators reach members through the manager. The method exists so the manager can fan out
uniformly over all seven subordinates.
