# src/xrGame/stalker_alife_task_actions.cpp

> The two operators that carry out what the alife simulation decided off-screen: hold station when there is nothing to do, and travel to the job a smart terrain has assigned.

**Needs** — [`stalker_alife_task_actions.h`](stalker_alife_task_actions.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_base_action.h`](stalker_base_action.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_human_brain.h`](../xrServerEntities/alife_human_brain.h.md) · [`alife_smart_terrain_task.h`](alife_smart_terrain_task.h.md) · [`game_location_selector.h`](game_location_selector.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — reached through its declarations in [`stalker_alife_task_actions.h`](stalker_alife_task_actions.h.md); callers name that, not this file.
**Tier floor** — T3: parameter assignment plus one two-level routing decision

## Purpose

This is the seam between the coarse world and the fine one. A stalker's *server object*
carries a smart-terrain assignment decided by the alife simulation while the stalker was
off-screen; when it comes online, something has to turn that assignment into footsteps.
These two operators are that something.

They implement the same three-phase lifecycle as every stalker operator, with the same
ordering rules — see [`stalker_alife_actions.cpp`](stalker_alife_actions.cpp.md).

## State

```text
RECORD ActionSolveZonePuzzle             # extends the stalker operator base
  stop_weapon_handling_time : int        # world clock; when to stow the weapon

RECORD ActionSmartTerrain                # extends the stalker operator base
  # no state; the assignment is re-read from the server object each cycle
```

## `CStalkerActionSolveZonePuzzle` — the idle stance

**Contract** — hold station: wander on the coarse graph, walking, upright, relaxed, looking
for cover, and stow the weapon after a randomized delay. Selected when the alife simulation
is running and no smart-terrain job is pending.

```text
FUNCTION initialize()
  base.initialize()
  stop_weapon_handling_time = now
  IF the stalker is currently holding its best weapon
    stop_weapon_handling_time = now + random in [30, 60) seconds

  desired direction = none
  path type         = game path          # coarse: no particular destination
  detail path type  = smooth
  body state        = standing
  movement type     = walk
  mental state      = free
  sight             = look for cover, not at anything

FUNCTION execute()
  base.execute()
  IF now >= stop_weapon_handling_time
    goal = strap the best weapon, or plain idle if it has none
  ELSE
    goal = idle while holding the best weapon

FUNCTION finalize()
  base.finalize()
  IF NOT alive  RETURN
  silence the sounds this behaviour started
```

**Notes** — this is the no-alife idle stance with the humming removed and the destination
left alone. The two exist separately because they answer opposite preconditions, and the
duplication is real: a rebuild can fold them into one operator parameterized by whether it
hums, and lose nothing. The name is vestigial in the same way the property it achieves is —
no zone puzzle is involved.

The randomized stow delay is the same decision as in the no-alife stance and for the same
reason: it desynchronizes a group of stalkers dropping into idle, and it stops a stalker
that has just left a fight from disarming on the same frame.

## `CStalkerActionSmartTerrain` — travel to the assigned job

**Contract** — route the stalker to the location its smart terrain assigned, across levels
if necessary. Selected when the alife simulation is running and an assignment is pending.
Carries thirty seconds of planner inertia. Hard-fails if the stalker's server object has a
smart terrain that hands back no task.

```text
FUNCTION initialize()
  base.initialize()
  desired direction = none
  # Coarse routing is asked for a MASKED selection rather than a random branch: the
  # stalker is going somewhere specific, so the route must be the route, not a plausible one.
  game route selection = masked
  detail path type = smooth
  body state       = standing
  movement type    = walk
  mental state     = free
  sight            = look along the path

  IF the stalker has no best weapon       goal = plain idle ; RETURN
  goal = plain idle
  IF the best weapon is already strapped  RETURN
  goal = idle while holding the best weapon

FUNCTION execute()
  base.execute()
  IF the plan has completed  goal = strap the best weapon
  play the humming sound with a 60-second period and a 10-second spread

  server = this stalker's server object              # the authoritative record
  REQUIRE server has a smart terrain assigned
  task = server.brain.smart_terrain.task(server)
  FAIL WITH "smart terrain is assigned but returns no task" IF task IS none

  # Two-level routing: cross the world first, then cross the level.
  IF the stalker is not on the task's game-graph vertex
    path type = game path
    game destination vertex = task.game_vertex
    RETURN

  path type = level path
  IF the task's navigation vertex is within this stalker's restrictions
    level destination vertex = task.level_vertex
    desired position         = task.position
    RETURN

  # Restricted away from the job: go as close as the restriction permits.
  move to the nearest accessible position to task.position
```

**Invariants**

- The coarse leg must finish before the fine leg begins. A stalker on a different level, or
  a different region of the game graph, has no level-graph route to the task at all, so
  asking for one would fail rather than degrade. The game-vertex comparison is the switch
  between the two.
- The assignment is re-read from the **server object** every cycle, not cached. The alife
  simulation can reassign a stalker while it is walking — a smart terrain fills up, a job is
  withdrawn — and the operator must follow. This is also why it reads the server side rather
  than the client side: the assignment is authoritative state, not simulation state.
- The restriction check is the live one: when the assigned location is outside the stalker's
  permitted space, it walks to the nearest position that is inside, rather than refusing or
  walking into a wall. That fallback is what keeps a badly placed job from freezing a
  stalker, and it is the main consumer of the nearest-legal-position search in
  [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md).

**Notes** — the thirty seconds of inertia set at construction is a planner hint, not a timer
this operator reads: it tells the brain to prefer staying with this operator for that long
rather than re-planning away from it. Travel to a smart terrain is a long action whose value
only appears at the end, so without the inertia a stalker would abandon a cross-level walk
the first cycle some other operator scored marginally higher, and oscillate.

Switching the coarse route selector to masked in `initialize` and back to random branching
in `finalize` is a global setting on the stalker's movement, not a per-request one, which is
why it has to be restored. Random branching is what makes idle stalkers wander along varied
routes; masked selection is what makes a stalker with a destination actually get there.

## Notes

A grenade-throwing test harness is compiled out throughout this file, replacing both
operators' bodies with "face the player and throw". It is development scaffolding, not
behaviour, and a rebuild should not carry it.
