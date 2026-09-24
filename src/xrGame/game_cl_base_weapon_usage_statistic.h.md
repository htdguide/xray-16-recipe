# src/xrGame/game_cl_base_weapon_usage_statistic.h

> Declares the match telemetry: every shot fired, every hit landed, where on the body it landed, what it was worth — implemented in [`game_cl_base_weapon_usage_statistic.cpp`](game_cl_base_weapon_usage_statistic.cpp.md) and persisted in [`..._save.cpp`](game_cl_base_weapon_usage_statistic_save.cpp.md).

**Needs** — [`game_cl_base_weapon_usage_statistic.cpp`](game_cl_base_weapon_usage_statistic.cpp.md) · [`game_cl_base_weapon_usage_statistic_save.cpp`](game_cl_base_weapon_usage_statistic_save.cpp.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`game_base_kill_type.h`](game_base_kill_type.h.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md)
**Used by** — [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`game_cl_base.cpp`](game_cl_base.cpp.md) · [`game_cl_base_weapon_usage_statistic.cpp`](game_cl_base_weapon_usage_statistic.cpp.md) · [`game_cl_base_weapon_usage_statistic_save.cpp`](game_cl_base_weapon_usage_statistic_save.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_cl_mp.cpp`](game_cl_mp.cpp.md) · [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T2: declares a record hierarchy with an explicit wire size per level

## Purpose

Declares the whole telemetry subsystem. Substance is in the two implementation twins.

The declaration's own decisions are the **record hierarchy** and the **three-team array
width**, and both are load-bearing.

The hierarchy is four levels deep and each level owns the one below: the collector owns a
player record per player, a player owns a weapon record per weapon he used, a weapon owns a
hit record per hit it landed. Every aggregate exists at every level, which is what lets a
report be produced per weapon, per player or per match without a second pass.

Every per-team array is **three** wide, not two, and the third slot is not a third team: the
mapping from the game's team numbering to this index is mode-dependent and is described in
the implementation twin. A rebuild that assumes two teams will mis-index deathmatch.

Exported units:

- `BulletData` — a projectile in flight, tracked from firing to removal: who fired it, from
  what, the projectile itself, how many hit reports it produced and how many the server has
  answered, and whether it has been removed.
- `victims_table` / `bone_table` — two compression dictionaries built per transmission; see
  the implementation twin.
- `HitData` — one hit: the segment it travelled, the bone, the target, the projectile, whether
  it was fatal, how many identical hits it stands for, and whether its fate is settled.
- `Weapon_Statistic` — per weapon: how many were bought, rounds and projectiles fired, hits
  and kills scored — each with a *delta* companion holding what has not yet been reported —
  plus explosion and bleed kills, a purchase histogram, and the hit list.
- `Player_Statistic` — per player: identity, total shots, per-team alive time, money earned,
  respawn count and objectives delivered, four special-kill counters, and the weapon list.
- `WeaponUsageStatistic` — the collector.
- `Bullet_Check_Request` / `Bullet_Check_Array` / `Bullet_Check_Respond_True` — the
  client-server exchange by which a client learns whether the hits it reported actually
  landed.

**Notes** — the purchase histogram is thirty-four buckets per team, and the bucket is chosen
from the buyer's money. The width and the bucketing rule are described where the rule lives;
thirty-four is "one bucket for under five hundred, then one per thousand" over the money
range the shipped modes allow.

Each serializable record declares its own **exact wire size** as a constant, used to decide
whether another record still fits in the packet. Those constants are hand-computed duplicates
of the serialization below them, and nothing checks the two agree. A rebuild should measure
rather than declare.

The collector holds a mutex, and the header carries a note asking for the dependency to be
removed. It cannot be: the telemetry is written from the network thread and read from the
simulation thread.
