# src/xrServerEntities/xrServer_Objects_ALife_Monsters.h

> Declares every record that is alive or is a zone: the trader mixin, the zone family, the creature levels, the human levels, and the squad.

**Needs** — [`xrServer_Objects_ALife.h`](xrServer_Objects_ALife.h.md) · [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`character_info_defs.h`](character_info_defs.h.md) · [`alife_movement_manager_holder.h`](alife_movement_manager_holder.h.md) · [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`alife_human_brain.h`](alife_human_brain.h.md)
**Used by** — [`Actor_Network.cpp`](../xrGame/Actor_Network.cpp.md) · [`CustomZone.cpp`](../xrGame/CustomZone.cpp.md) · [`Entity.cpp`](../xrGame/Entity.cpp.md) · [`InventoryOwner.cpp`](../xrGame/InventoryOwner.cpp.md) · [`TorridZone.cpp`](../xrGame/TorridZone.cpp.md) · [`ZoneVisual.cpp`](../xrGame/ZoneVisual.cpp.md) · [`actor_mp_server.cpp`](../xrGame/actor_mp_server.cpp.md) · [`actor_mp_server.h`](../xrGame/actor_mp_server.h.md) · [`ai_rat.cpp`](../xrGame/ai/monsters/rats/ai_rat.cpp.md) · [`monster_state_smart_terrain_task_inline.h`](../xrGame/ai/monsters/states/monster_state_smart_terrain_task_inline.h.md) · [`alife_anomalous_zone.cpp`](../xrGame/alife_anomalous_zone.cpp.md) · [`alife_combat_manager.cpp`](../xrGame/alife_combat_manager.cpp.md) · [`alife_creature_abstract.cpp`](../xrGame/alife_creature_abstract.cpp.md) · [`alife_group_abstract.cpp`](../xrGame/alife_group_abstract.cpp.md) · _and 49 more_
**Tier floor** — T2: a hierarchy declaration; the layouts are in the implementation.

## Purpose

Declares the first of the two concrete-record families. Layouts and contracts are in
[`xrServer_Objects_ALife_Monsters.cpp`](xrServer_Objects_ALife_Monsters.cpp.md).

Despite the file's name, it holds **two** unrelated families that share it only because both
descend from the restrictor and dynamic-visual levels: the **zones** (anomalies, which are
restrictor volumes that hurt things) and the **creatures**. The split is arbitrary and a
rebuild is free to separate them.

Three organizing ideas govern the declarations.

**"Trades and has an identity" is a mixin, not a level.** The actor, the trader and every
human share a set of facts — money, a character profile, a faction, a rank, a reputation,
a name — that has nothing to do with being alive. It is mixed in, and it is mixed into the
actor *alongside* being a creature, which is why the actor inherits from three places.

**"Is scheduled offline" is also a mixin.** A creature the alife simulation advances while
nobody is looking carries a scheduling mixin and a movement-position mixin (see
[`alife_movement_manager_holder.h`](alife_movement_manager_holder.h.md)) on top of being a
creature. A crow is a creature and is *not* scheduled.

**The brain is composed, not inherited.** A monster record owns a brain; a human record owns
a richer one. The choice is made by a single overridable construction step rather than by
the hierarchy, which is what lets a human be a monster with a different head — see
[`alife_monster_brain.h`](alife_monster_brain.h.md) and
[`alife_human_brain.h`](alife_human_brain.h.md).

## Exported units

**The identity mixin**

- `CSE_ALifeTraderAbstract` — money, a character profile, the resolved individual, a faction
  index, rank, reputation, display name, portrait, corpse-looting permissions, and the
  "infinite ammunition" flag. Also the profile-resolution algorithm, which is the most
  consequential thing in the family.
- `CSE_ALifeTrader` — a dynamic visual object that is a trader and nothing else: a shopkeeper
  with no body to simulate.

**The zones**

- `CSE_ALifeCustomZone` — a restrictor volume with a power, a damage type, an owner, and an
  on/off duty cycle.
- `CSE_ALifeAnomalousZone` — adds the artefact-spawning behaviour and the offline
  interaction radius.
- `CSE_ALifeTorridZone` — an anomaly that follows an authored motion path.
- `CSE_ALifeZoneVisual` — an anomaly with a model and an attack animation.

**The creatures**

- `CSE_ALifeCreatureAbstract` — the level that is *alive*: health, killer, team/squad/group,
  orientation, dynamic restrictions, the three evaluation-function type numbers, and the
  death timestamp.
- `CSE_ALifeMonsterAbstract` — adds scheduling, offline movement, the brain, immunity
  factors, the smart-terrain assignment and the rank.
- `CSE_ALifeCreatureActor` — the player: a creature, a trader and a ragdoll at once.
- `CSE_ALifeCreatureCrow`, `CSE_ALifeCreaturePhantom` — creatures that are deliberately
  excluded from the navigation graph and from online/offline switching.
- `CSE_ALifeMonsterRat`, `CSE_ALifeMonsterZombie` — the two records whose tuning lives in
  the record instead of in the configuration section. See the implementation for why this is
  a mistake worth not repeating.
- `CSE_ALifeMonsterBase` — every other creature: a monster with a ragdoll and a
  special-object reference.
- `CSE_ALifePsyDogPhantom` — a monster that never becomes the simulation's active choice.

**The humans**

- `CSE_ALifeHumanAbstract` — a trader *and* a monster, with the human brain.
- `CSE_ALifeHumanStalker` — adds a ragdoll and a starting dialogue.

**The squad**

- `CSE_ALifeOnlineOfflineGroup` — a record whose state is a set of member identities, which
  moves and thinks as one thing while offline and dissolves into its members online.

## Notes

**The header opens by suppressing a macro-redefinition diagnostic for its whole length** and
restoring it at the end. The clash is between the hit-type names and something in the
platform headers; it is incidental, but it is a signal that the hit-type vocabulary in
[`alife_space.h`](alife_space.h.md) uses names generic enough to collide.

**A comment in the source calls the mixin arrangement a way "to prevent virtual
inheritance"**, and every mixin consequently declares a pair of methods answering "which
record am I part of". That is a language workaround. The decision it encodes is real and
must survive: **a mixin must be able to reach the record it is mixed into**, because
profile resolution sets the record's visual and the record's team.
