# src/xrGame/ai/monsters/poltergeist/poltergeist.h

> Declares the poltergeist — an invisible flying creature that never touches its target — together with the two interchangeable abilities it attacks through and the shared base those abilities extend.

**Needs** — [`poltergeist.cpp`](poltergeist.cpp.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../telekinesis.h`](../telekinesis.h.md) · [`../energy_holder.h`](../energy_holder.h.md)
**Used by** — [`HUDTarget.cpp`](../../../HUDTarget.cpp.md) · [`poltergeist.cpp`](poltergeist.cpp.md) · [`poltergeist_ability.cpp`](poltergeist_ability.cpp.md) · [`poltergeist_flame_thrower.cpp`](poltergeist_flame_thrower.cpp.md) · [`poltergeist_movement.cpp`](poltergeist_movement.cpp.md) · [`poltergeist_script.cpp`](poltergeist_script.cpp.md) · [`poltergeist_state_attack_hidden_inline.h`](poltergeist_state_attack_hidden_inline.h.md) · [`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md) · [`poltergeist_telekinesis.cpp`](poltergeist_telekinesis.cpp.md) · [`script_game_object_use2.cpp`](../../../script_game_object_use2.cpp.md)
**Tier floor** — T2: owns particle emitters, sounds and physics handles whose release order matters

## Purpose

Declares the surfaces implemented across
[`poltergeist.cpp`](poltergeist.cpp.md), [`poltergeist_ability.cpp`](poltergeist_ability.cpp.md),
[`poltergeist_flame_thrower.cpp`](poltergeist_flame_thrower.cpp.md) and
[`poltergeist_telekinesis.cpp`](poltergeist_telekinesis.cpp.md).

The poltergeist is the creature that breaks the chapter's usual shape. It has no melee, no
body to collide with while hidden, no navigation on the ground; it floats at a height it
re-randomises as it drifts, it is invisible almost all the time, and it attacks the player
through one of two abilities chosen by a single word in its configuration section.

Four types are declared here.

## `Poltergeist`

The creature. On top of the shared creature base it composes two mixins — telekinesis (the
ability to lift and throw physics objects) and an energy budget (the drain-while-hidden
scalar) — and adds three things of its own: **a hidden mode** in which its physical body is
destroyed and only a particle effect remains, **a drifting height**, and **a detection level**
that is the real gate on everything it does.

Its exported surface, beyond the usual creature lifecycle:

- `ability` — the one special ability this poltergeist was built with.
- `is_hidden`, `enable_hiding` / `disable_hiding` — the hidden mode and the lock that pins it.
- `detected_enemy` — whether the detection level has passed the threshold at which the creature
  starts circling.
- `set_actor_ignore` / `actor_ignore` — the script switch that makes it inert towards the player.
- `fly_around_distance`, `fly_around_change_direction_time` — the circling parameters its
  attack state reads.
- `physical_impulse(position)`, `strange_sounds(position)` — the two ambient scare effects.
- `update_height` — re-randomises the drift target.
- `current_position` — the creature's position *on the navigation graph*, which while hidden is
  not its rendered position. See [`poltergeist_movement.cpp`](poltergeist_movement.cpp.md).
- `run_home_point_when_enemy_inaccessible` — answers **no**, overriding the creature base: a
  poltergeist whose target is unreachable does not retreat home, because nothing is unreachable
  to something that floats.

## `PolterAbility`

The base of the two abilities, and a real interface rather than a convenience: it owns the
particle emitters and the idle sound that every poltergeist has regardless of which ability it
carries, and it declares the six moments an ability may react to — `load`, `update_schedule`,
`update_frame`, `on_hide`, `on_show`, `on_destroy`, `on_die`, `on_hit`. Implemented in
[`poltergeist_ability.cpp`](poltergeist_ability.cpp.md).

## `PolterFlame`

The flamethrower ability: spawns columns of fire at points near the target rather than near
itself. Implemented in
[`poltergeist_flame_thrower.cpp`](poltergeist_flame_thrower.cpp.md), which also documents a
loaded-but-unused scanner sub-ability.

## `PolterTele`

The telekinetic ability: lifts nearby physics objects and hurls them at the player. Implemented
in [`poltergeist_telekinesis.cpp`](poltergeist_telekinesis.cpp.md).
