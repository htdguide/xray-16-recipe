# src/xrGame/ai/monsters/burer/burer.h

> Declares the burer: a creature that fights with telekinesis, a travelling gravity wave and a bullet shield, with about thirty authored numbers behind those three abilities.

**Needs** — [`base_monster.h`](../basemonster/base_monster.h.md) · [`telekinesis.h`](../telekinesis.h.md) · [`anim_triple.h`](../anim_triple.h.md) · [`scanning_ability.h`](../scanning_ability.h.md) · [`burer.cpp`](burer.cpp.md)
**Used by** — [`burer.cpp`](burer.cpp.md) · [`burer_fast_gravi.cpp`](burer_fast_gravi.cpp.md) · [`burer_script.cpp`](burer_script.cpp.md) · [`burer_state_attack_antiaim_inline.h`](burer_state_attack_antiaim_inline.h.md) · [`burer_state_attack_gravi_inline.h`](burer_state_attack_gravi_inline.h.md) · [`burer_state_attack_inline.h`](burer_state_attack_inline.h.md) · [`burer_state_attack_run_around_inline.h`](burer_state_attack_run_around_inline.h.md) · [`burer_state_attack_shield_inline.h`](burer_state_attack_shield_inline.h.md) · [`burer_state_attack_tele_inline.h`](burer_state_attack_tele_inline.h.md) · [`burer_state_manager.cpp`](burer_state_manager.cpp.md) · [`script_game_object_use2.cpp`](../../../script_game_object_use2.cpp.md)
**Tier floor** — T2: a creature class that owns physics-facing abilities

## Purpose

Declares the surface implemented in [`burer.cpp`](burer.cpp.md). It is the widest creature header in the chapter because the burer's three abilities each carry their own parameter block, and the behaviour states read those blocks directly rather than through accessors — the class is, deliberately, the shared mutable state between the creature and its attack tree.

The creature mixes in the telekinesis ability, which owns the list of objects currently held aloft and their per-object state machines.

## `Burer`

Beyond the base lifecycle points it declares:

- **The gravity wave.** `GraviObject`, an embedded record for the single travelling wave the creature may have in flight, plus `UpdateGraviObject` to advance it and `StartGraviPrepare` / `StopGraviPrepare` for the wind-up effect drawn on the victim.
- **The shield.** `ActivateShield`, `DeactivateShield`, `CanDeactivateShieldEarly`, and `need_shotmark`, which suppresses bullet decals while the shield is up.
- **Telekinesis presentation.** `StartTeleObjectParticle` / `StopTeleObjectParticle`, the visual marking of a held object.
- **Facing.** `face_enemy`, the shared "turn toward the enemy unless already roughly aimed, then stand" step every burer attack sub-state calls.
- **The stamina drain.** `StaminaHit`, the private callback the anti-aim ability fires — see the implementation.
- `get_force_gravi_attack` / `set_force_gravi_attack`, a script-settable override that makes the next gravity attack bypass its own preconditions.
- `ability_distant_feel` — answers yes: the burer senses at range, which is what lets it start a fight before it has line of sight.
- `get_monster_class_name` — returns `"burer"`.

## Parameter records

```text
RECORD GraviObject                       # the single wave in flight
  active          : bool
  current_pos     : position
  from_pos        : position             # where it was launched
  target_pos      : position             # where it is heading
  time_last_update: int
  victim          : reference to the entity it was launched at

RECORD GraviParams                       # all authored, see burer.cpp
  speed, step, radius, min_dist, max_dist : real
  time_to_hold    : int
  cooldown        : int
  impulse_to_objects, impulse_to_enemy, hit_power : real
```

Loose authored fields for telekinesis (object count, mass window, search radius, distance window, raise speed, fly speed, hold height, maximum duration), for the shield (duration, cooldown, particle and its period), and for the anti-aim stamina drain (weight-to-stamina factor, weapon-drop factor and velocity) and the run-away geometry (runaway distance, normal distance, maximum runaway time).

## Sound identifiers

The creature extends the shared creature-sound enumeration with two of its own, for the gravity and telekinetic attacks. Extending rather than replacing is the pattern: a creature's custom sounds start at the shared enumeration's "custom" mark so that the shared sound player keeps working unchanged.
