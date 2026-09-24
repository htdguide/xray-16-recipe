# src/xrGame/ai/monsters/controller/controller.h

> Declares the controller creature, implemented in [`controller.cpp`](controller.cpp.md).

**Needs** — [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../controlled_actor.h`](../controlled_actor.h.md) · [`../anim_triple.h`](../anim_triple.h.md)
**Used by** — [`controlled_entity.h`](../controlled_entity.h.md) · [`controlled_entity_inline.h`](../controlled_entity_inline.h.md) · [`controller.cpp`](controller.cpp.md) · [`controller_animation.cpp`](controller_animation.cpp.md) · [`controller_direction.cpp`](controller_direction.cpp.md) · [`controller_psy_hit.cpp`](controller_psy_hit.cpp.md) · [`controller_script.cpp`](controller_script.cpp.md) · [`controller_state_attack_fast_move_inline.h`](controller_state_attack_fast_move_inline.h.md) · [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md) · [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md) · [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md) · [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md) · [`controller_state_control_hit_inline.h`](controller_state_control_hit_inline.h.md) · [`controller_state_manager.cpp`](controller_state_manager.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CController`. Substance is in [`controller.cpp`](controller.cpp.md).

The declaration itself carries one decision worth naming: the class is a base monster **and**
an actor-hold handler at once. Inheriting the hold rather than owning one is how the question
"which creature is holding the player" is answered without a registry — the set-piece attack
casts its own creature to the hold interface and installs it.

## State

Described in the implementation twin. The public fields are the aura radius and damage, the
set-piece attack element, the eight sounds, the two extra gaits, the thrall list, the mental
state, the friendly-community override list, and the four set-piece tuning values — all
public because the creature's states and its own control elements read them directly.

Exported units:

- **lifecycle** — `Load`, `reload`, `reinit`, `UpdateCL`, `shedule_Update`, `Die`,
  `net_Spawn`, `net_Destroy`, `net_Relcase`.
- **control assembly** — `create_base_controls` (substitutes this creature's own animation
  and direction drivers), `CheckSpecParams`, `InitThink` (merges the thralls' enemy
  memories), `TranslateActionToPathParams`, `head_orientation`,
  `ability_pitch_correction` (refused: this creature stays upright),
  `use_center_to_aim` (yes), `run_home_point_when_enemy_inaccessible` (no).
- **enthralment** — `HasUnderControl`, `TakeUnderControl`, `UpdateControlled`,
  `FreeFromControl`, `OnFreedFromControl`, `set_controlled_task`.
- **psi bolt** — `psy_fire`, `can_psy_fire`, `draw_fire_particles`,
  `set_psy_fire_delay_zero`, `set_psy_fire_delay_default`.
- **set piece** — `tube_fire`, `can_tube_fire`, `tube_ready`, `get_tube_min_distance`.
- **control hit** — `control_hit`, `play_control_sound_start`, `play_control_sound_hit`.
- **relations** — `is_relation_enemy`, `load_friend_community_overrides`,
  `is_community_friend_overrides`.
- **mental state** — the two-value enumeration and `set_mental_state`.
- **hits** — `HitEntity`, which drains the actor's stamina and forces a drop.
- **accessors** — `custom_anim`, `custom_dir`, reaching this creature's own drivers at their
  derived type rather than the base one. Every creature-specific state in this family goes
  through these.
- **script surface** — a registration hook; see
  [`controller_script.cpp`](controller_script.cpp.md).

`TakeUnderControl` is declared and defined nowhere; enthralment happens inside
`UpdateControlled`. `get_monster_class_name` answers the literal `controller`, which is how
script and diagnostics name this creature.
