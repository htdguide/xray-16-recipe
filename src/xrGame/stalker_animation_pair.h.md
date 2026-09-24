# src/xrGame/stalker_animation_pair.h

> Declares one animation channel — the thing that remembers what a body part is playing — implemented in [`stalker_animation_pair.cpp`](stalker_animation_pair.cpp.md).

**Needs** — [`stalker_animation_pair.cpp`](stalker_animation_pair.cpp.md) · [`stalker_animation_pair_inline.h`](stalker_animation_pair_inline.h.md) · [`ai/ai_monsters_anims.h`](ai/ai_monsters_anims.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_pair.cpp`](stalker_animation_pair.cpp.md) · [`stalker_animation_pair_inline.h`](stalker_animation_pair_inline.h.md) · [`stalker_animation_script.cpp`](stalker_animation_script.cpp.md) · [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md)
**Tier floor** — T2: holds a handle into the renderer's active-blend list

## Purpose

Declares `CStalkerAnimationPair`. A stalker's animation manager owns four of these — legs,
torso, head, script — plus one for whole-body animations. Each is a *channel*: it remembers
which motion that body part is supposed to be playing, whether the renderer has actually
been told, and which blend the renderer handed back.

Exported units:

- `reset` — forget everything; the channel is idle and considered up to date.
- `animation(motion)` / `animation()` — set and read the wanted motion; setting a
  different motion is what marks the channel stale.
- `actual()` / `make_inactual()` — has the wanted motion been started.
- `select(variants, weights)` — pick one motion out of a variant list, sticky per list.
- `play(...)` — start the wanted motion if the channel is stale.
- `blend()` — the renderer's handle for the running motion, or none.
- `synchronize(other)` — copy another channel's playback time into this one.
- `step_dependence(bool)` / `step_dependence()` — does this channel drive footsteps.
- `global_animation(bool)` / `global_animation()` — is this a whole-body animation.
- `add_callback` / `remove_callback` / `callback` / `need_update` — the end-of-animation
  subscriber list.
- `on_animation_end` — invoked by the renderer when the motion finishes.
- `callback_on_collision(bool)` and its reader — should a collision also end this motion.
- `target_matrix()` in three forms — where a root-moving animation should end up.
- `use_animation_movement_control(skeleton, motion)` — does this motion move the root.

Substance in [`stalker_animation_pair.cpp`](stalker_animation_pair.cpp.md).

**Notes** — the header carries a compile-time switch that decides whether whole-body
animations address bone parts individually or always cover all four. Only the
individually-addressed form is ever built; the other is dead. A rebuild should keep the
bone-part mask as a plain parameter and drop the switch.
