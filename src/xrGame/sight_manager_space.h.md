# src/xrGame/sight_manager_space.h

> The closed set of ways a creature can be told where to look.

**Needs** — _(none)_
**Used by** — [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`script_game_object_script.cpp`](script_game_object_script.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md) · [`script_watch_action.h`](script_watch_action.h.md) · [`script_watch_action_inline.h`](script_watch_action_inline.h.md) · [`script_watch_action_script.cpp`](script_watch_action_script.cpp.md) · [`sight_action.cpp`](sight_action.cpp.md) · [`sight_action.h`](sight_action.h.md) · [`sight_action_inline.h`](sight_action_inline.h.md) · [`sight_manager.cpp`](sight_manager.cpp.md) · [`sight_manager.h`](sight_manager.h.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md)
**Tier floor** — T4: an enumeration

## Purpose

One enumeration, in its own header so that the sight action, the sight manager, the script
look order and the stalker brain can all name a sight type without pulling in each other's
declarations. That is the only reason for the file.

## State

`Stateless.`

## The sight type

```text
ENUM SightType
  current_direction      # hold whatever direction the head already has
  path_direction         # look along the next leg of the movement path
  direction              # look along a given world direction
  position               # look at a given world point
  object                 # look at an entity, tracking it
  cover                  # face the direction that exposes least of the creature
  search                 # cover, tilted up, for scanning
  look_over              # (declared; no behaviour of its own)
  cover_look_over        # face cover, but glance aside periodically
  fire_object            # look at an entity with the torso turned to it, aim-quality
  fire_position          # look at a point with the torso turned to it
  animation_direction    # let the playing animation decide; the head follows the body
  dummy                  # the all-ones sentinel for "no sight type"
```

**Invariants** — `fire_position` is marked in the source as *must be removed*, and the
sight action's construction path silently rewrites it: a look order built with it becomes
`position` with the torso-look flag set. So the two are the same behaviour under two
names, and the name survives only because scripts use it (it is exported as `fire_point`).
A rebuild should implement `position` plus a torso flag and keep `fire_position` as an
alias at the script boundary.

`look_over` has no execution branch anywhere; only `cover_look_over` does. It is reachable
only as a value, never as a behaviour, and can be dropped.

The sentinel value is the all-ones pattern, so the enumeration's storage must be an
unsigned 32-bit width — it is packed into saved and scripted state at that width.
