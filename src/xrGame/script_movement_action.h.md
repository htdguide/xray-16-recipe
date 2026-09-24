# src/xrGame/script_movement_action.h

> Declares the movement channel of a scripted action: where an entity should go, in what posture, by what kind of path.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path_params.h`](../xrAICore/Navigation/PatrolPath/patrol_path_params.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_movement_action.cpp`](script_movement_action.cpp.md) · [`script_movement_action_inline.h`](script_movement_action_inline.h.md) · [`script_movement_action_script.cpp`](script_movement_action_script.cpp.md)
**Tier floor** — T2: a record plus a tagged union over goal kinds

## Purpose

The busiest channel of a [scripted action](script_entity_action.h.md), and the only one
that is really a **tagged union**: a movement order names exactly one kind of goal — an
object to reach, a patrol path to walk, a position to path to, a position to go to
*without* pathing, a navigation vertex, a set of held vehicle controls, a jump target, or a
leader to follow — and the remaining goal fields are then dead. The tag is not set
directly; it is a side effect of whichever setter the script called last, which is the one
thing about this type a rebuilder must not miss.

It also serves two different consumers with one record: humanoid entities read the posture,
movement type and path type, while monsters read a single coarse *move action* and a speed
parameter instead. A given channel is built for one or the other, and nothing marks which.

## State

```text
RECORD ScriptMovementAction EXTENDS ActionChannel   # the base supplies `completed`
  goal_type        : enum { object, patrol_path, path_position, no_path_position,
                            path_node_position, input, jump_to_position,
                            follow_leader, none }
  # --- goal payload; which fields are live is decided by goal_type ---
  object_to_go     : optional<ClientObject>   # goal_type = object
  patrol_path      : optional<PatrolPath>     # goal_type = patrol_path
  path_name        : text                     # the patrol path's authored name, kept for
                                              # save/load: the path object is not persistable
  patrol_start     : enum { nearest, ... }    # which point of the path to begin at
  patrol_stop      : enum { continue, ... }   # what to do on reaching the end
  patrol_random    : bool                     # default true: pick the next point at random
                                              # rather than in authored order
  previous_point   : int                      # resume point, carried from the path params
  destination      : vector                   # the three position-flavoured goal types
  node_id          : int                      # goal_type = path_node_position: the
                                              # navigation vertex the position lies on
  input_keys       : bitset                   # goal_type = input: held vehicle controls
  # --- humanoid parameters ---
  body_state       : enum { stand, crouch }
  movement_type    : enum { stand, walk, run }
  path_type        : enum { smooth, smooth_dodge, smooth_criteria }
  speed            : real                     # zero means "the entity's own default"
  # --- monster parameters ---
  move_action      : enum { walk_fwd, walk_bkwd, run_fwd, drag, jump, steal,
                            walk_with_leader, run_with_leader }
  speed_param      : enum { default, force }  # force overrides the creature's own gait speed
  dist_to_end      : real                     # −1 means "all the way"; otherwise stop this
                                              # far short of the goal
  jump_factor      : real                     # arc tuning for a jump goal
```

**Invariants**

- `goal_type` is written by the setters, never by the script. Setting a patrol path makes
  the goal a patrol path; setting a position makes it a pathed position; setting input keys
  makes it a control order. The last such call wins, and there is no way to clear the tag
  back to none.
- The tag `none` is the constructed default and means the channel orders no movement.
- `path_name` is redundant with `patrol_path` while the level is loaded and is *not*
  redundant across a save: the path object is a level-lifetime thing and the name is what
  survives.
- The monster fields (`move_action`, `speed_param`, `dist_to_end`) and the humanoid fields
  (`body_state`, `movement_type`, `path_type`, `speed`) are never both meaningful. Which
  set is live is decided by which constructor built the channel and recorded nowhere — a
  rebuild is free to split this into two channel types, and probably should.
- Every mutating call clears the channel's completion flag; see
  [`script_movement_action_inline.h`](script_movement_action_inline.h.md).

## Exported units

Constructors, grouped by consumer and documented in
[`script_movement_action.cpp`](script_movement_action.cpp.md) and
[`script_movement_action_inline.h`](script_movement_action_inline.h.md):

- default — orders nothing.
- humanoid: (posture, movement type, path type) plus one of an object, a patrol path
  parameter block, or a position, with an optional speed.
- direct: (position, speed) — the goal that deliberately skips the pathfinder.
- vehicle: (held keys) with an optional speed.
- monster: (move action) plus one of a position, a patrol path parameter block, an object,
  or a (vertex, position) pair, with an optional stop distance and an optional speed
  parameter.

Setters: `set_body_state`, `set_movement_type`, `set_path_type`, `set_object_to_go`,
`set_patrol_path`, `set_position`, `set_speed`, `set_patrol_start`, `set_patrol_stop`,
`set_patrol_random`, `set_input_keys`, `initialize`.

**Notes**

The vehicle control bits are a bitset rather than an enum because a driving order holds
several at once — forward and left, say — and two of the bits (engine on, engine off) are
not directions at all but momentary commands. A rebuild should keep the flag layout, since
scripts combine the exported constants arithmetically.

`node_id` and `destination` are both given for a navigation-vertex goal because the vertex
names a *cell* and the position names where within it: a creature told to go to a vertex
alone would arrive at its centre, which is visibly wrong on a wide cell.
