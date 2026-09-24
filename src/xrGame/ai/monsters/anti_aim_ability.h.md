# src/xrGame/ai/monsters/anti_aim_ability.h

> Declares the ability that punishes a player for holding a weapon steady on a creature: it builds up a detection level and, when full, throws the player's aim off.

**Needs** — [`anti_aim_ability.cpp`](anti_aim_ability.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`control_combase.h`](control_combase.h.md)
**Used by** — [`anti_aim_ability.cpp`](anti_aim_ability.cpp.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`burer.cpp`](burer/burer.cpp.md) · [`burer_state_attack_antiaim_inline.h`](burer/burer_state_attack_antiaim_inline.h.md)
**Tier floor** — T2: a per-update integrator plus camera-effector lifetime bookkeeping

## Purpose

Declares the surface implemented in [`anti_aim_ability.cpp`](anti_aim_ability.cpp.md).

## Exported units

- **the ability** — a control component with its own settings block, a detection level it
  integrates each update, and the camera effector it launches.
- **settings** — timeout between uses, the named camera effectors to choose from, a freeze
  time, the maximum aim angle that still counts as "aimed at me", and separate gain and
  decay speeds for the detection level.
- **hit callback** — a notification the owning creature installs, fired at the moment the
  ability lands, so the creature can play a sound or apply an effect of its own.
- **death notification** — tears the ability down when its creature dies, because the
  effector it launched outlives the creature otherwise.
