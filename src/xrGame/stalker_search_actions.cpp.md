# src/xrGame/stalker_search_actions.cpp

> The three actions of the lost-enemy search: walk to his last known place, take a spot that
> watches it, and sit there until he is written off.

**Needs** — [`stalker_search_actions.h`](stalker_search_actions.h.md) · [`stalker_combat_action_base.h`](stalker_combat_action_base.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_evaluators.h`](cover_evaluators.h.md) · [`cover_point.h`](cover_point.h.md) · [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md)
**Used by** — [`stalker_search_actions.h`](stalker_search_actions.h.md)
**Tier floor** — T2: movement, sight and cover requests driven from remembered perception.

## Purpose

Each of the three operators declared by the search planner is one of these actions. They
share a single structure — a lifecycle of initialize, execute-per-cycle, finalize — and one
mechanism that matters more than any of them individually: the **hit interrupt**. Whichever
search action is running, being shot at mid-search aborts the search and hands control back
to open combat.

The lifecycle order is load-bearing and identical in all three: `initialize` fixes the
creature's gait, posture and mental state and snapshots the hit clock; `execute` runs each
cycle and may set a world property the planner reads; `finalize` releases whatever
`initialize` claimed.

## State

Each action holds two things.

```text
RECORD SearchAction
  combat_state  : reference to the combat planner's world state   # written on interrupt
  last_hit_time : int (ms)   # the hit clock as of this action's initialize
```

**Invariant** — `last_hit_time` is a *watermark*, not a timestamp of interest. It is set
once at initialize and compared, never advanced. A hit recorded after it is by definition a
hit taken during this action.

## The hit interrupt

Identical in all three `execute` bodies, and it runs before anything else the action does:

```text
FUNCTION check_interrupt() -> bool           # true means "abandon this cycle"
  hit := memory.hit.hit(memory.enemy.selected)
  IF hit EXISTS AND hit.time > last_hit_time
    combat_state.LookedOut      := false
    combat_state.PositionHolded := false
    combat_state.EnemyDetoured  := false
    RETURN true
  RETURN false
```

It clears three properties in the **parent combat planner's** world state, not in the search
planner's. That is the whole trick: the search planner's own goal is unreachable except
through its last operator, so a search cannot end itself. Falsifying facts the parent relied
on invalidates the parent's plan, the parent re-plans on the next cycle, and the search
branch is abandoned from above. A creature shot at while searching stops searching and
fights — and it does so through the planner rather than through a special case.

The three properties cleared are exactly the ones that mean "I have settled": I have leaned
out, I am holding a position, I have finished circling. Clearing them is the statement *the
situation is no longer settled*.

Every `execute` also returns early when the enemy has fallen out of memory entirely, because
all three actions are steering toward a remembered position and there is nothing to steer
toward.

## `ReachEnemyLocation`

**Contract** — walk to the level position where the enemy was last remembered, looking at
that position on the way, and report arrival by setting `EnemyLocationReached`.

**Initialize** sets the creature's whole movement posture in one block, and the combination
is the behaviour: no fixed facing, route planned on the level navigation mesh, smooth
detail path, mental state *danger*, standing, walking. A stalker searching walks — it does
not run and does not crouch — with its weapon ready but not raised at anything. It also
releases its squad cover reservation, for the same reason the planner does: a moving
creature must not hold a cover slot.

```text
FUNCTION execute()
  IF check_interrupt() OR enemy_not_remembered()
    RETURN

  IF NOT movement.path_completed
    look_at(remembered.position + half_a_unit_up)
    RETURN

  # arrived at the previous goal; aim the route at the remembered cell
  IF movement.accessible(remembered.vertex)
    movement.destination_vertex := remembered.vertex
  ELSE
    movement.destination := nearest_accessible_to(remembered.position, remembered.vertex)

  look_at(remembered.position + half_a_unit_up)

  IF movement.path_completed                       # still complete: the target is where we stand
    state.EnemyLocationReached := true
    play_start_search_sound()
```

**Invariants** — the destination is re-aimed every cycle in which the path is complete, not
once at initialize. That is what lets the creature chase a *moving* memory: as the enemy is
glimpsed elsewhere, the remembered position moves and the creature re-targets without
re-planning. The doubled completion test is the arrival condition: the path is complete
*and* re-targeting at the current memory did not produce a new path, i.e. there is nowhere
further to walk.

**Notes** — the half-unit lift on the look-at target aims at a body rather than at the
floor, so the creature's head points where a person would be. The inaccessible-position
fallback matters because a remembered position can be inside geometry or outside the
creature's restrictors: the creature goes as close as its movement space allows rather than
failing to move at all.

The action announces itself with a search sound on arrival, suppressed in the silent-combat
build. A rebuild should keep it — it is the player's only cue that a stalker has lost them
and is now hunting, and it is what makes hiding legible.

## `ReachAmbushLocation`

**Contract** — from the enemy's last known position, pick a cover point that watches it,
walk there, and report arrival by setting `AmbushLocationReached`.

```text
FUNCTION execute()
  IF check_interrupt() OR enemy_not_remembered()
    RETURN

  ambush_evaluator.setup(enemy_remembered_position, my_remembered_position, 10)
  point := best_cover(near enemy_remembered_position, radius 10, ambush_evaluator, my_restrictions)
  IF point IS none
    ambush_evaluator.setup(same arguments)
    point := best_cover(near enemy_remembered_position, radius 30, ambush_evaluator, my_restrictions)

  IF point EXISTS
    movement.destination_vertex := point.vertex
    movement.destination        := point.position
  ELSE
    movement.destination        := nearest_accessible_position()

  IF NOT movement.path_completed
    play_enemy_lost_sound()
    RETURN

  state.AmbushLocationReached := true
```

**Invariants** — the search is re-run every cycle, so the ambush spot tracks the memory as
it moves. The cover query is always centred on the *enemy's* remembered position, not on the
creature: the creature is looking for somewhere that watches that place, which is why the
cover evaluator is given both positions.

**Notes** — the ten-then-thirty widening is the load-bearing part. Ten world units is the
distance at which an ambush is an ambush; thirty is the fallback that keeps the creature
from standing in the open when the good spots near the target are taken or unreachable. The
evaluator is re-configured before the second query with identical arguments, which is
redundant as written but harmless.

The "enemy lost" sound plays every cycle the creature is still walking, not once. That reads
as a creature muttering while it searches, and it is deliberate enough that the shipped
build guards it with the same silent-combat switch as the other combat barks.

An alternative implementation of this same cover search sits disabled inside
`ReachEnemyLocation`; the two were evidently one action once, and splitting them is what
made the ambush step re-plannable on its own.

## `HoldAmbushLocation`

**Contract** — crouch in the chosen spot, watch over the cover, and after the action's own
completion condition is met, stop treating the enemy as worth remembering.

**Initialize** crouches the creature and points its gaze over the edge of its cover — the
"look over cover" sight mode, which is a peek rather than a fixed stare.

```text
FUNCTION execute()
  IF check_interrupt() OR enemy_not_remembered()
    RETURN
  IF NOT completed()
    RETURN                                    # still inside the action's inertia window
  IF remembered.last_seen_time + 60000 >= now
    RETURN                                    # the memory is still fresh; keep waiting
  memory.disable(enemy)                       # write him off
```

**Invariants** — two independent clocks must both expire. The action's own completion — the
fifteen-second inertia the planner attached to this operator — says *I have waited long
enough to call this an ambush*. The sixty-second memory age says *the sighting is stale*. A
creature that took up an ambush moments after losing a very recent contact keeps waiting
past its inertia window, because the memory is not yet a minute old.

Disabling the enemy in memory is what ends the search: with the enemy no longer a live
memory, the enemy properties fall to false all the way up the tree, the combat branch's
preconditions stop holding, and the creature returns to whatever it was doing. The search
planner's own goal — *no longer a pure enemy* — is claimed by this operator, but the real
mechanism is this one line, and a rebuild that leaves it out produces a stalker that crouches
behind a crate forever.

**Notes** — sixty seconds is the enemy-memory horizon used here and nowhere else in this
file; it is written out rather than named. It is long enough that the player cannot simply
break line of sight and walk back in.
