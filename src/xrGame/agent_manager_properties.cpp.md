# src/xrGame/agent_manager_properties.cpp

> The three squad evaluators: each answers one yes/no question by polling the roster for any member whose own perception already selected something.

**Needs** — [`agent_manager_properties.h`](agent_manager_properties.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_manager_space.h`](agent_manager_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`agent_manager_properties.h`](agent_manager_properties.h.md)
**Tier floor** — T3: three existential scans over a small list.

## Purpose

The squad's world state is not sensed independently — the squad has no eyes. Each of these
evaluators is an *any* over the roster: the squad believes something is true when at least
one member's individual perception has already selected it. That is the whole design
decision this file encodes, and it is why the squad brain can run once a second: it reads
conclusions, never raw perception.

## `EvaluatorItem`

**Contract** — True when any member of the roster has an item selected by its own item
perception. Allocates nothing; cost is linear in roster size, which is single digits.

## `EvaluatorEnemy`

**Contract** — True when any member of the **combat** subset of the roster has an enemy
selected. Non-combat members are excluded, so a wounded or scripted-out member cannot put
the squad into combat.

## `EvaluatorDanger`

**Contract** — True when any member of the roster has a danger selected.

```text
FUNCTION evaluate(roster_subset, which_perception) -> bool
  FOR EACH member IN roster_subset
    IF member.object.memory.(which_perception).selected() IS PRESENT
      RETURN true
  RETURN false
```

**Notes** — The three differ only in which roster subset and which of the member's three
perception channels they read; a rebuild should write one parameterized evaluator. They
are three types here because the planner registers evaluators by type identity.

This file is compiled in the script-aware translation unit, which is incidental: the
evaluator base type is exported to Lua so scripts can add their own evaluators to this
planner's space.
