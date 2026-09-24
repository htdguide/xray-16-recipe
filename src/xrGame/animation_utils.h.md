# src/xrGame/animation_utils.h

> Declares the bone-freeze record and the ancestry query implemented in [`animation_utils.cpp`](animation_utils.cpp.md).

**Needs** — [`animation_utils.cpp`](animation_utils.cpp.md)
**Used by** — [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`animation_utils.cpp`](animation_utils.cpp.md) · [`character_hit_animations.cpp`](character_hit_animations.cpp.md) · [`death_anims_predicates.cpp`](death_anims_predicates.cpp.md) · [`imotion_position.cpp`](imotion_position.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`animation_utils.cpp`](animation_utils.cpp.md). The
freeze record is a plain aggregate meant to be embedded by value in whatever subsystem owns
the frozen bone.

Exported units:

- **`anim_bone_fix`** — bone, parent and the frozen parent-relative offset, with
  **`fix`** (capture and install), **`refix`** (reinstall without recapturing),
  **`release`** (uninstall) and **`deinit`** (uninstall and forget).
- **`find_in_parents`** — whether one bone is an ancestor of another, root excluded.
