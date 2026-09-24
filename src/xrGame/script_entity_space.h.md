# src/xrGame/script_entity_space.h

> Names the six channels a scripted action can drive, plus the retirement event.

**Needs** — _(none)_
**Used by** — [`patrol_path_manager.cpp`](patrol_path_manager.cpp.md) · [`script_entity.cpp`](script_entity.cpp.md) · [`script_entity.h`](script_entity.h.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md)
**Tier floor** — T2: a constant set

## Purpose

The enumeration that identifies which part of a scripted action a completion callback is
reporting about. It is a separate, dependency-free file because both the action machinery
and the script-visible facade need the names without needing each other.

## `ActionType`

**Contract** — a dense enumeration whose *values* reach script (they are passed as the
second argument of every action callback) and are therefore frozen.

```text
ENUM ActionType
  movement   = 0
  watch
  animation
  sound
  particle
  object
  count            # the number of real channels; used to size arrays
  removed          # not a channel: "the action finished and left the queue"
```

**Notes**

`count` sits *inside* the enumeration and before `removed`, so it doubles as both the array
size and a sentinel. `removed` therefore has the value `count + 1`, which means a rebuild
that separates the two concepts must keep `removed`'s numeric value, since shipped scripts
compare against it.
