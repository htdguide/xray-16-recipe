# src/xrGame/stalker_animation_state.h

> Declares the per-body-state bundle of loaded animations implemented in [`stalker_animation_state.cpp`](stalker_animation_state.cpp.md).

**Needs** — [`stalker_animation_state.cpp`](stalker_animation_state.cpp.md) · [`stalker_animation_names.h`](stalker_animation_names.h.md) · [`ai/ai_monsters_anims.h`](ai/ai_monsters_anims.h.md) · [`stalker_animation_state_inline.h`](stalker_animation_state_inline.h.md)
**Used by** — [`stalker_animation_data.cpp`](stalker_animation_data.cpp.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`stalker_animation_state.cpp`](stalker_animation_state.cpp.md) · [`stalker_animation_state_inline.h`](stalker_animation_state_inline.h.md) · [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md)
**Tier floor** — T3: a tree of resolved motion handles

## Purpose

Declares `CStalkerAnimationState`: everything a stalker can play *while in one body state*
(crouched, normal, or damaged-normal). One of these exists per body state, and the three of
them together with the head and no-weapon sets make up a stalker's full animation data.

The type is built entirely out of the name-driven collection templates, so its shape is
its own documentation: each nesting level is one fragment table from
[`stalker_animation_names.h`](stalker_animation_names.h.md).

Exported units:

- `CStalkerAnimationState` — the bundle. Four members:
  - the **global** set, indexed by the global-name table (damage, escape, critical hits,
    panic);
  - the **torso** set, a two-level nest: animation slot, then weapon action;
  - the **movement** set, a two-level nest: walk/run, then direction;
  - the **in-place** set, a flat list of the ten stationary leg animations.
- `Load(skeleton, base_name)` — resolves every name against a model's motion bank.

The substance — what the nesting means and why the in-place set is owned separately — is in
[`stalker_animation_state.cpp`](stalker_animation_state.cpp.md).
