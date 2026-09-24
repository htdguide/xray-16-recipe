# src/xrGame/stalker_animation_data.h

> Declares the three animation tables a stalker model's motions are resolved into, loaded together and shared between every stalker using that model.

**Needs** — [`stalker_animation_state.h`](stalker_animation_state.h.md) · [`stalker_animation_names.h`](stalker_animation_names.h.md)
**Used by** — [`stalker_animation_data.cpp`](stalker_animation_data.cpp.md) · [`stalker_animation_data_storage.cpp`](stalker_animation_data_storage.cpp.md) · [`stalker_animation_global.cpp`](stalker_animation_global.cpp.md) · [`stalker_animation_head.cpp`](stalker_animation_head.cpp.md) · [`stalker_animation_legs.cpp`](stalker_animation_legs.cpp.md) · [`stalker_animation_manager.cpp`](stalker_animation_manager.cpp.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md)
**Tier floor** — T3: a declaration over three indexed tables

## Purpose

Declares the surface implemented in [`stalker_animation_data.cpp`](stalker_animation_data.cpp.md).

## Exported units

- `m_part_animations` — indexed by body state; each entry holds that state's torso animations by weapon and action, its movement animations by movement type and direction, its in-place animations, and its whole-body animations.
- `m_head_animations` — a flat table of head motions.
- `m_global_animations` — indexed by weapon kind; whole-body actions performed with that weapon.
- Constructor — loads all three from one skeleton.

## Notes

The three tables are public fields rather than accessors, and every selection site in the
animation system indexes them directly with a literal. That coupling is real: the literals
are positions in the word lists these tables were generated from, so the lists and the
selection sites must change together. A rebuild that keeps the flat tables should at least
name the indices.
