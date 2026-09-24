# src/xrGame/script_movement_action.cpp

> The movement channel's constructors that decide something: unpacking a patrol path parameter block, deriving a goal kind from a monster move action, and the pathless goal.

**Needs** — [`script_movement_action.h`](script_movement_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_params.h`](../xrAICore/Navigation/PatrolPath/patrol_path_params.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

The constructors that could not be inline, which turns out to be exactly the ones that do
more than assign: those that flatten a patrol path parameter block into the channel, the
one that derives a goal kind from the monster move action, and the one that deliberately
bypasses the pathfinder. The rest are in
[`script_movement_action_inline.h`](script_movement_action_inline.h.md).

## Patrol path goals

**Contract** — three constructors take a *patrol path parameter block* — the script-side
object that names a path and how to traverse it — and flatten its five fields into the
channel. They differ only in which consumer's parameters accompany them.

```text
FUNCTION adopt_patrol(params)
  set_patrol_path(params.path, params.path_name)   # also sets goal_type = patrol_path
  set_patrol_start(params.start_type)
  set_patrol_stop(params.stop_type)
  set_patrol_random(params.random)
  previous_point = params.previous_index
```

**Invariants**

- The block is *copied*, not referenced. A script may reuse and mutate one parameter block
  to build several orders, and shipped scripts do.
- `previous_index` is carried across so a patrol resumed after an interruption — a fight,
  a save and load — starts from where the entity left off rather than from the nearest
  point.

**Notes**

The parameter block exists because a patrol order has five settings and a constructor
taking five positional arguments would be unreadable from script. It is the only channel
argument that is itself a script-visible object.

## Monster goal derived from the move action

**Contract** — the monster constructor taking a position reads the move action to decide
the goal kind, because two of the eight move actions are not "walk to a place" at all:

```text
FUNCTION monster_goal_from(move_action, position, stop_distance)
  move_action  = move_action
  set_position(position)              # tentatively goal_type = path_position
  speed_param  = default
  dist_to_end  = stop_distance        # −1 means "all the way"
  IF move_action IS jump THEN
    goal_type = jump_to_position      # a ballistic arc, not a path
  ELSE IF move_action IS walk_with_leader OR run_with_leader THEN
    goal_type = follow_leader         # the position is a hint; the goal is another creature
```

**Notes**

The derivation is why this constructor cannot be a plain field copy, and it is the one
place the two vocabularies — goal kind and move action — are related. A rebuild that keeps
them separate must reproduce this mapping, since a script asking a creature to jump passes
the jump move action and a destination and expects an arc.

## Navigation-vertex goal

**Contract** — the monster constructor taking a vertex and a position writes both, tags
the goal as a vertex goal, takes the default speed parameter, and clears the completion
flag directly rather than through a setter.

**Notes**

This is the only constructor that writes the destination field without going through the
position setter, precisely so the setter does not overwrite the vertex tag with the
position tag. The ordering dependency is invisible and fragile; a rebuild should set the
tag last, once, from the constructor that knows the intent.

## Pathless goal

**Contract** — the (position, speed) constructor sets a standing posture, a standing
movement type and a smooth path type, writes the destination, and then tags the goal
*no path position*.

**Invariants** — the tag is written after the position setter, overriding the tag that
setter left. That is the whole point: this goal asks the entity to head at the destination
directly, ignoring the navigation mesh, which is how scripted set-pieces move an entity
through a doorway the pathfinder would refuse.

## `set_object_to_go`

**Contract** — takes a game object facade or nothing, stores the client object behind it
(or nothing), tags the goal as an object goal, and clears the completion flag. Passing
nothing still tags the goal as an object goal, leaving a channel that will never complete —
which is a usable idiom for "stop here until I say otherwise", and a common script bug.
