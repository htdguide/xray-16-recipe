# src/xrGame/ai_monster_space.h

> The shared vocabulary of creature behaviour: posture, gait, mental state, the actions a creature can perform with a held object, and the animation categories scripts steer monsters by.

**Needs** — _(none)_
**Used by** — [`controller_direction.h`](ai/monsters/controller/controller_direction.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`script_animation_action.h`](script_animation_action.h.md) · [`script_monster_action.h`](script_monster_action.h.md) · [`script_monster_hit_info_script.cpp`](script_monster_hit_info_script.cpp.md) · [`script_movement_action.cpp`](script_movement_action.cpp.md) · [`script_movement_action.h`](script_movement_action.h.md) · [`script_movement_action_script.cpp`](script_movement_action_script.cpp.md) · [`script_object_action.h`](script_object_action.h.md) · [`script_sound_action.h`](script_sound_action.h.md) · [`sight_control_action.h`](sight_control_action.h.md) · [`smart_cover_action.h`](smart_cover_action.h.md) · _and 6 more_
**Tier floor** — T3: enumerations and one small record.

## Purpose

Almost every AI file in the game names a body state, a movement type or an object action.
They live here, in one namespace with no dependencies, so that a header can use the word
without pulling in a creature. Several of these enumerations are also exported to the
script layer, which freezes both their names and their values.

## State

```text
ENUM MentalState          # how alarmed the creature is; drives gait, aim and animation set
  Danger = 0              #   alert, weapon ready
  Free                    #   relaxed
  Panic                   #   fleeing

ENUM BodyState
  Crouch = 0
  Stand
  Dummy  = -1

ENUM MovementType
  Walk = 0
  Run
  Stand

ENUM MovementDirection    # strafe axis, independent of facing
  Forward = 0
  Backward
  Left
  Right
```

```text
ENUM ObjectAction         # what a creature is doing with the item in its hands
  # the engine's own set, in this order:
  Switch1, Switch2                   # select fire modes
  Reload1, Reload2
  Aim1, Aim2
  Fire1, FireNoReload, Fire2
  Idle
  Strapped                           # slung, not in hands
  Drop
  AimReady1, AimReady2               # aimed and holding
  AimForceFull1, AimForceFull2       # aimed with the full aiming animation forced
  # reachable only from scripts:
  Activate, Deactivate, Use, TurnOn, TurnOff
  # reachable only from the superseded object handler:
  Show, Hide, Take, Misfire1, Empty1
  NoItems                            # "nothing in hands"
  Dummy = -1
```

**Invariants** — `NoItems` is not the next value in sequence: it is `Idle` with every bit of
a 16-bit width set beneath it, which places it far above every real action while still
deriving from `Idle`. The effect is that code testing "is the creature idle" by masking
also treats empty hands as idle, while code switching on the exact value distinguishes
them. A rebuild must keep the two facts — empty-hands is idle-like, and it is a distinct
value — however it encodes them.

The "1" and "2" suffixes are the two weapon slots, not two variants of an action: the
creature's hands are modelled as two slots and most item actions name which one.

```text
RECORD BoneRotation       # one bone being aimed at a target, e.g. the head or the spine
  current : rotation      # where it points now
  target  : rotation      # where it should point
  speed   : real          # radians per second it may turn toward the target
```

```text
# Script-facing monster control. These are the words a Lua script uses to drive a
# non-human creature directly, bypassing the planner.
ENUM ScriptMonsterMoveAction
  WalkFwd, WalkBkwd, Run, Drag, Jump, Steal, WalkWithLeader, RunWithLeader

ENUM ScriptMonsterSpeedParam
  Default = 0, ForceSpeed, None = -1     # ForceSpeed overrides the gait's own speed

ENUM ScriptMonsterAnimAction
  StandIdle, CapturePrepare, SitIdle, LieIdle, Eat, Sleep, Rest,
  Attack, LookAround, Turn, NoAction = -1

ENUM ScriptMonsterGlobalAction
  Rest = 0, Eat, Attack, Panic, None = -1

ENUM ScriptSoundAnim
  Custom = 0, Default        # whether a sound brings its own animation

ENUM MonsterHeadAnimType     # facial expression during dialogue
  Normal = 0, Angry, Glad, Kind, None = -1
```

**Notes** — `Drag` and `Steal` are creature-specific gaits (dragging a corpse, moving
stealthily) that exist as movement actions rather than as separate animation states
because the script surface addresses movement by one identifier.

The "dummy" and "none" values are uniformly the maximum of the width rather than a
sentinel like -1 in a signed sense; the enumerations are unsigned throughout because they
are packed into save records and script calls at a fixed width.
