# src/xrGame/moving_bones_snd_player.h

> Declares the per-bone motion sound player implemented in [`moving_bones_snd_player.cpp`](moving_bones_snd_player.cpp.md).

**Needs** — [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`PhysicObject.cpp`](PhysicObject.cpp.md) · [`moving_bones_snd_player.cpp`](moving_bones_snd_player.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `moving_bones_snd_player`, which plays a looping sound whose pitch tracks how fast
one bone of an animated model is rotating — the servo whine of a moving turret, the creak
of a swinging door. Substance is in
[`moving_bones_snd_player.cpp`](moving_bones_snd_player.cpp.md).

Exported units:

- `moving_bones_snd_player` — holds the watched bone, the sound handle, the bone's previous
  world transform, the smoothed angular speed and the three tuning numbers read from
  configuration.
- `update` — the per-frame step: measure, smooth, start or stop the loop, set pitch and
  position.
- `play` / `stop` — start and stop the loop explicitly.
- `is_active` — always answers true; see the note in the implementation twin.
- `create_moving_bones_snd_player(object)` — the free factory: builds one for a game object
  if its configuration declares the feature, and otherwise nothing.
- `is_active(player)` — the null-tolerant form of the query, so callers holding an absent
  player need no test of their own.
