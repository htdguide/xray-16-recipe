# src/xrGame/Entity.h

> Declares the damageable, killable, team-affiliated entity implemented in [`Entity.cpp`](Entity.cpp.md).

**Needs** — [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`damage_manager.h`](damage_manager.h.md) · [`EntityCondition.h`](EntityCondition.h.md)
**Used by** — [`Car.cpp`](Car.cpp.md) · [`Car.h`](Car.h.md) · [`Entity.cpp`](Entity.cpp.md) · [`Explosive.cpp`](Explosive.cpp.md) · [`Grenade.cpp`](Grenade.cpp.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`PDA.cpp`](PDA.cpp.md) · [`Spectator.h`](Spectator.h.md) · [`WeaponFire.cpp`](WeaponFire.cpp.md) · [`phantom.h`](ai/phantom/phantom.h.md) · [`base_client_classes_wrappers.h`](base_client_classes_wrappers.h.md) · [`entity_alive.cpp`](entity_alive.cpp.md) · _and 5 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the layer that adds health, team membership and death to a physical object.
Substance in [`Entity.cpp`](Entity.cpp.md).

Exported units:

- `CEntity` — a physics-shell holder that also carries a per-bone damage scaling table.
- `GetfHealth`, `SetfHealth`, `GetMaxHealth`, `SetMaxHealth`, `g_Alive` — health, and the
  fact that "alive" is nothing but health above zero.
- `Hit` — the damage path: impulse, then health, then the reaction with the health
  actually lost, then the base.
- `CalcCondition` — apply damage to health, floored at -1000 so overkill is
  representable.
- `HitSignal`, `HitImpulse` — **what this class demands of a subclass**: react to being
  hit, and respond physically to an impulse.
- `KillEntity`, `Die`, `OnEvent` — record the killer once, broadcast a death event, and
  apply the death when it returns.
- `killer_id`, `AlreadyDie`, `set_death_time`, `GetLevelDeathTime`, `GetGameDeathTime` —
  the killer record and the two death clocks, one real and one in-world.
- `id_Team`, `id_Squad`, `id_Group`, `g_Team`, `g_Squad`, `g_Group` — the three-level
  affiliation.
- `ChangeTeam`, `on_before_change_team`, `on_after_change_team` — move between
  affiliations atomically: unregister, assign, register.
- `Load`, `reload`, `reinit`, `net_Spawn`, `net_Destroy`, `shedule_Update`, `_construct` —
  the lifecycle; the registry membership must mirror the registration flag exactly.
- `create_entity_condition` — the factory a subclass overrides to install a richer
  condition than health alone.
- `SEntityState` — the movement state a subclass may publish: jumping, crouching, falling,
  sprinting, plus linear and angular speed.
- `g_State`, `g_fireParams`, `g_stateFire`, `IsVisibleForHUD`, `in_solid_state` — small
  predicates subclasses answer.
- `IsFocused`, `IsMyCamera` — the entity the player controls; the entity the player looks
  through. They differ in vehicles and scripted sequences.
- `set_ready_to_save` — flush before a save; empty in the base.
- `m_fMorale`, `m_fFood`, `m_dwBodyRemoveTime` — morale (a compiled-in constant), a food
  level, and how long a corpse stays on the level.
