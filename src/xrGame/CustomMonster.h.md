# src/xrGame/CustomMonster.h

> Declares the thinking-creature base implemented in [`CustomMonster.cpp`](CustomMonster.cpp.md), and names the five capabilities every creature composes.

**Needs** — [`entity_alive.h`](entity_alive.h.md) · [`script_entity.h`](script_entity.h.md) · [`xrEngine/Feel_Vision.h`](../xrEngine/Feel_Vision.h.md) · [`xrEngine/Feel_Sound.h`](../xrEngine/Feel_Sound.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`trajectories.h`](trajectories.h.md) · [`CustomMonster_inline.h`](CustomMonster_inline.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomMonster_VCPU.cpp`](CustomMonster_VCPU.cpp.md) · [`CustomMonster_inline.h`](CustomMonster_inline.h.md) · [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`ai_rat.h`](ai/monsters/rats/ai_rat.h.md) · [`ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai_trader.h`](ai/trader/ai_trader.h.md) · [`danger_manager.cpp`](danger_manager.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`hit_memory_manager.cpp`](hit_memory_manager.cpp.md) · [`item_manager.cpp`](item_manager.cpp.md) · _and 9 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares a creature as the composition of a living entity, a script-controllable entity,
and the three senses — vision, sound and touch — each of which is a separate interface the
creature opts into. That composition is the architectural statement: a sense is not a
property of a creature, it is an interface the creature implements, and something without
eyes simply does not implement vision. Substance in
[`CustomMonster.cpp`](CustomMonster.cpp.md).

## The two abstract operations

A concrete creature must supply exactly two things:

- `Think` — one decision cycle. Everything the brain does hangs off this.
- `SelectAnimation` — given a facing, a movement direction and a speed, choose what to
  play.

Everything else has a base implementation.

## Exported units

**Lifecycle**

- `_construct`, `net_Spawn`, `net_Destroy`, `Load`, `reload`, `reinit`, `save`, `load` —
  the subsystems are built before the base constructs; memory is reloaded before anything
  can consult it; memory is persisted only while alive.
- `create_memory_manager`, `create_movement_manager`, `create_sound_visitor` — the
  factories a species overrides to substitute its own.

**The two ticks**

- `shedule_Update` — the thinking tick: trim the state queue, sense, remember, think, act,
  publish a state.
- `UpdateCL` — the presentation tick: interpolate a pose from the queue and apply it.
- `UpdatePositionAnimation` — drive the physical movement toward the interpolated position
  and pick an animation from the result.

**The network state queue**

- `net_update` — one timestamped pose: model yaw, torso rotation, position, health.
- `net_update::lerp` — angle-aware interpolation of two states.
- `NET`, `NET_Last`, `NET_Time`, `NET_WasInterpolating`, `NET_WasExtrapolating` — the
  queue, the applied pose and the mode flags.
- `net_Export`, `net_Import` — publish the newest queued state; append a received one if
  it is newer than the tail.

**Vision**

- `eye_bone`, `eye_matrix`, `eye_fov`, `eye_range`, `m_tEyeShift`, `m_fEyeShiftYaw` — the
  eye's bone, frame, cone and per-species offset.
- `Exec_Visibility`, `eye_pp_s0`, `eye_pp_s1`, `eye_pp_s2` — the two-tick split: build the
  frame and query the frustum; then raycast.
- `update_range_fov` — modulate sight range by the current weather, per species.
- `feel_visible_isRelevant` — the pre-filter: only living things on other teams.
- `feel_vision_mtl_transp` — how much a given material lets through.
- `visual_memory` — the creature's visual memory manager.

**The other senses**

- `feel_sound_new` — a sound reached me; hand it to sound memory unless dead.
- `feel_touch_contact`, `feel_touch_on_contact` — overridden so that an anomaly counts as
  touched only when the creature's origin is inside it.
- `dcast_FeelSound` — the downcast the sound system needs.

**Movement and pose**

- `movement`, `memory`, `sound`, `sound_user_data_visitor` — the subsystem accessors.
- `Orientation`, `head_orientation` — the body's and the head's rotation.
- `PitchCorrection` — lean the body to the slope of the navigation cell underfoot.
- `mk_orientation`, `mk_rotation` — build a facing from a direction, in the horizontal
  plane only and in full respectively.
- `Exec_Action`, `Exec_Look` — the per-tick action and look streams.
- `predict_position`, `target_position`, `spatial_move`, `spatial_sector_point`,
  `get_moving_object` — the spatial surface.
- `create_anim_mov_ctrl`, `destroy_anim_mov_ctrl`, `ForceTransform` — handing the
  transform to an animation and taking it back, and teleporting.
- `angle_lerp_bounds`, `vfNormalizeSafe` — see
  [`CustomMonster_inline.h`](CustomMonster_inline.h.md).

**Damage**

- `Hit` — dropped entirely while invulnerable.
- `HitSignal`, `Die` — the reaction hook and death.
- `invulnerable` — the flag.
- `update_critical_wounded`, `critically_wounded`, `critical_wound_type`,
  `critical_wounded_state_stop` — the accumulate-and-threshold mechanism for losing the
  use of a limb.
- `load_critical_wound_bones`, `critical_wound_external_conditions_suitable`,
  `critical_wounded_state_start` — what it demands of a species: a bone-to-wound-type
  table, a veto, and the state itself.
- `load_killer_clsids`, `is_special_killer` — the per-species list of classes whose kills
  are treated specially.

**Evaluation**

- `useful` / `evaluate`, three overloads each — is this item, enemy or danger worth caring
  about, and how much. The memory managers call back into these, so overriding one changes
  how the creature's own memory ranks things.

**Miscellaneous**

- `SAnimState` — a four-way directional animation set (forward, back, left strafe, right
  strafe), resolved from one base name.
- `panic_threshold` — the health fraction below which a creature panics.
- `on_enemy_change`, `on_restrictions_change` — notifications.
- `human_being`, `is_base_monster_with_enemy`, `should_wait_to_use_corspe_visual`,
  `get_custom_pitch_speed`, `visual_name` — small species predicates.
- `UsedAI_Locations` — true: creatures occupy navigation positions.
