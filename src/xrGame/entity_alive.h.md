# src/xrGame/entity_alive.h

> Declares the base of every creature that can be hurt, implemented in [`entity_alive.cpp`](entity_alive.cpp.md).

**Needs** — [`Entity.h`](Entity.h.md) · [`entity_alive_inline.h`](entity_alive_inline.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`Wound.h`](Wound.h.md) · [`monster_community.h`](monster_community.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`material_manager.h`](material_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`Actor.h`](Actor.h.md) · [`AmebaZone.cpp`](AmebaZone.cpp.md) · [`Artefact.cpp`](Artefact.cpp.md) · [`BastArtifact.cpp`](BastArtifact.cpp.md) · [`BastArtifact.h`](BastArtifact.h.md) · [`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md) · [`BottleItem.cpp`](BottleItem.cpp.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`GraviZone.cpp`](GraviZone.cpp.md) · [`HUDTarget.cpp`](HUDTarget.cpp.md) · _and 32 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the living-entity base class. Substance is in
[`entity_alive.cpp`](entity_alive.cpp.md).

Its own load-bearing content is what the class **demands of a subclass** — three pure
virtuals that every creature must answer and that between them define what it means to be a
creature here:

- `g_WeaponBones` — which three bones carry a weapon (left hand, and two right-hand
  attachment points). Required because the weapon placement code is shared and cannot guess.
- `ffGetFov` and `ffGetRange` — the creature's field of view and sight range. Required
  because perception is shared and every creature sees differently.

The other thing the declaration fixes is the **ownership split**: the condition model and
the material manager are owned here and destroyed here; the wound lists are *borrowed* from
the condition model; and the blood and fire presentation tables are shared across every
living entity in the process, loaded once from configuration by whichever creature is
constructed first.

Exported units:

- `CEntityAlive` — the class.
- `cast_entity_alive` — the capability query by which the rest of the game recognizes a
  creature.
- `Load` / `reload` / `reinit` / `_construct` — configuration and two-phase construction.
- `save` / `load` / `net_SaveRelevant` — serialisation; the condition model's state follows
  the base entity's, and every creature is save-relevant.
- `net_Spawn` / `net_Destroy` / `net_Relcase` — the lifecycle, including re-deriving blood
  and fire effects from restored wounds.
- `shedule_Update` — the per-update pass, and the one place death is detected.
- `Hit` / `HitImpulse` / `CalcCondition` — damage. The impulse is disabled.
- `Die` — the ordered death sequence.
- `conditions` / `material` / `create_entity_condition` — the two owned subsystems.
- `g_Radiation` / `SetfRadiation` — the fraction-to-percentage boundary.
- `tfGetRelationType` / `is_relation_enemy` / `monster_community` — faction relations, by
  species.
- `BloodyWallmarks` / `PlaceBloodWallmark` / `LoadBloodyWallmarks` / `UnloadBloodyWallmarks`
  / `ClearBloodWounds` — blood on the world.
- `StartBloodDrops` / `UpdateBloodDrops` — dripping from open wounds.
- `StartFireParticles` / `UpdateFireParticles` / `LoadFireParticles` / `UnloadFireParticles`
  — burning wounds.
- `PHGetSyncItemsNumber` / `PHGetSyncItem` / `PHFreeze` / `PHUnFreeze` / `PHGetLinearVell` /
  `ph_sound_player` / `character_ik_controller` / `get_collision_hit_callback` /
  `set_collision_hit_callback` — delegation to the character physics support, with the
  live/dead split between character controller and ragdoll.
- `create_anim_mov_ctrl` / `destroy_anim_mov_ctrl` / `OnChangeVisual` — animation-driven
  movement brackets, and the hit-bone cache invalidation.
- `get_new_local_point_on_mesh` / `get_last_local_point_on_mesh` /
  `fill_hit_bone_surface_areas` — where on this body something may land, and how that point
  follows the animation.
- `predict_position` / `target_position` — hooks; the base answers with the current position.
- `set_lock_corpse` / `is_locked_corpse` — the corpse-contention lock with its release
  cool-down.
- `visual_memory` — absent at this level; creatures that see override it.
- `ef_creature_type` / `ef_weapon_type` / `ef_detector_type` — world-state classifications.
- `OnHitHealthLoss` / `OnCriticalHitHealthLoss` / `OnCriticalWoundHealthLoss` /
  `OnCriticalRadiationHealthLoss` — empty notification hooks, one per way of losing health,
  so a subclass can react to *how* it is dying and not merely that it is.
- `human_being` — false here; the discriminator between people and mutants, used by the
  "walk past a weak monster" rule and by several others.
- `is_agresive` / `is_start_attack` / `m_bMobility` / `m_fAccuracy` / `m_fIntelligence` /
  `m_squad_index` — behaviour flags and tunables read by the planner.
