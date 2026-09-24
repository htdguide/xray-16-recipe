# src/xrGame/searchlight.h

> Declares the searchlight entity: a scripted object whose visual carries a steerable spot light and glow.

**Needs** — [`script_object.h`](script_object.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md)
**Used by** — [`script_game_object_use.cpp`](script_game_object_use.cpp.md) · [`searchlight.cpp`](searchlight.cpp.md)
**Tier floor** — T1: holds renderer light and glow handles with a defined release order

## Purpose

Declares the surface implemented in [`searchlight.cpp`](searchlight.cpp.md).

## Exported units

- **The class** — a scripted object, so it accepts the same script entity actions as a
  creature does, but implements only two of them.
- **Lifecycle** — load from a configuration section, spawn from a server record, scheduled
  update, per-frame update.
- **`used_ai_locations`** — answers *no*: a searchlight occupies no navigation vertex.
- **`assign_watch` and `assign_object`** — the two script entity actions it honours:
  aim the beam, and switch it on or off.
- **`current_direction`** — where the beam points right now, as a unit vector; read by
  scripts and by anything testing whether the beam falls on something.
- **Private steering internals** — turn on/off, the two bone callbacks, and the target
  setter; all described in the implementation twin.

## State

```text
RECORD Searchlight
  brightness    : real                # cached intensity of the authored colour
  colour_anim   : optional<LightAnim>  # named animation driving colour over time
  light, glow   : render handles
  guide_bone    : bone id             # the bone whose transform is the beam's origin
  bone_yaw      : (bone id, angular speed)
  bone_pitch    : (bone id, angular speed)
  start         : (yaw, pitch)        # orientation at spawn — the sweep's reference frame
  current       : (yaw, pitch)
  target        : (yaw, pitch)
```
