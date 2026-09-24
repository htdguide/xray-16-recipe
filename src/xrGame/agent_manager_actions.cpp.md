# src/xrGame/agent_manager_actions.cpp

> The four squad-level actions, each a thin shell whose real work is telling the right subordinate managers to run their distribution passes this cycle.

**Needs** — [`agent_manager_actions.h`](agent_manager_actions.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`agent_corpse_manager.h`](agent_corpse_manager.h.md) · [`agent_explosive_manager.h`](agent_explosive_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md)
**Used by** — [`agent_manager_actions.h`](agent_manager_actions.h.md)
**Tier floor** — T3: three lifecycle hooks per action.

## Purpose

An operator in this engine's planner is not a pure state transition — it is an object with
a three-phase lifecycle (`initialize` when it becomes the current action, `execute` on
every cycle it remains current, `finalize` when it is displaced). The squad's four
operators use that lifecycle to switch expensive per-cycle work on and off: a distribution
pass runs only while the squad is in the mode that needs it.

## `NoOrders`

**Contract** — The quiet mode. Does nothing while current; on being displaced, clears the
corpse manager's accumulated state. Leaving the quiet mode is exactly the moment stale
corpse interest becomes wrong.

## `GatherItems`

**Contract** — The looting mode. No hooks at all — the item selection each member already
made is the whole behaviour, and the squad only needs to be *in* this mode so that combat
and danger modes are excluded.

## `KillEnemy`

**Contract** — The combat mode. On entry, releases every claimed combat position, so that
positions are re-claimed against the new fight rather than inherited from the previous
one. On each cycle: distributes enemies over the combat members, makes members react to
explosives threatening them, and makes them react to a member's death. Takes no action on
exit.

```text
FUNCTION KillEnemy.initialize()
  location.clear()          # forget position claims from the previous situation

FUNCTION KillEnemy.execute()
  enemy.distribute_enemies()      # assign each combat member a target
  explosive.react_on_explosives() # flee the grenade, even mid-fight
  corpse.react_on_member_death()  # a squadmate falling changes the plan
```

**Notes** — The exit hook is deliberately empty. An earlier version redistributed enemies
on exit; the surviving comment marks it as removed, and the reason is visible in the
order: distribution on exit would assign targets to a squad that is about to stop
fighting, and the assignment would then persist into the quiet mode.

## `ReactOnDanger`

**Contract** — The danger mode, entered when something threatens the squad but no enemy is
selected. On entry, releases claimed combat positions for the same reason combat does. On
each cycle, runs the explosive reaction and the member-death reaction — the same two
passes combat runs, minus enemy distribution, because there is no enemy to distribute.
