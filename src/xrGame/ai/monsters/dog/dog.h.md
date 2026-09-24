# src/xrGame/ai/monsters/dog/dog.h

> Declares the blind dog, implemented in [`dog.cpp`](dog.cpp.md).

**Needs** — [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../controlled_entity.h`](../controlled_entity.h.md) · [`dog.cpp`](dog.cpp.md)
**Used by** — [`dog.cpp`](dog.cpp.md) · [`dog_script.cpp`](dog_script.cpp.md) · [`dog_state_manager.cpp`](dog_state_manager.cpp.md) · [`monster_enemy_memory.cpp`](../monster_enemy_memory.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the dog as a base creature that is additionally *controllable* — a controller can take it
over — and exposes the numbered-animation machine's fields publicly, because the pack states in
[`group_states/`](../group_states/README.md) drive them directly rather than through methods. That
direct coupling is the reason the group states are dog-shaped even though they are templates.

## `CAI_Dog`

The creature lifecycle it overrides:

- **Load / reload** — build the animation vocabulary and read tuning; see the implementation twin
- **reinit** — reset the animation machine, register jump data, arm the home wander band
- **UpdateCL** — per-frame update, with the out-of-band brain run on animation completion

The behaviour hooks it overrides:

- **CheckSpecParams** — one-off animation flourishes requested by a state
- **HitEntityInJump** — deliver the mid-leap bite with the animation's own parameters
- **check_start_conditions** — the leader-only jump rule
- **get_attack_rebuild_time** — distance-scaled path rebuild interval
- **can_use_agressive_jump** — the enemy-is-above test
- **ability_can_drag** — yes; dogs drag corpses
- **get_monster_class_name** — `"dog"`, the identity the script and debug layers use

The numbered-animation surface, driven from the group states:

- **set_current_animation(n)** / **get_number_animation** — request and read the clip index
- **random_anim** — pick an idle clip, weighted by the section's animation factor and the clock
- **start_animation** — capture the animation channel and play the requested clip
- **anim_end_reinit** — force-release the channel mid-clip
- **get_custom_anim_state** / **set_custom_anim_state** — is a vocabulary clip playing
- **is_night** — the world-clock test the idle picker uses

Its public fields — the appetite, pack and sleep bookkeeping plus the five authored timeouts — are
listed as a record in the implementation twin. The private half is the animation machine's own
state and the wander band.

Also declared: the script registration hook (see [`dog_script.cpp`](dog_script.cpp.md)) and a
debug-only key handler.
