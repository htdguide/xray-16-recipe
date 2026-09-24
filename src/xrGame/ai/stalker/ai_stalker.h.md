# src/xrGame/ai/stalker/ai_stalker.h

> Declares the stalker: the human brain, and the widest single class in the game.

**Needs** — [`CustomMonster.h`](../../CustomMonster.h.md) · [`object_handler.h`](../../object_handler.h.md) · [`AI_PhraseDialogManager.h`](../../AI_PhraseDialogManager.h.md) · [`step_manager.h`](../../step_manager.h.md) · [`ai_stalker_inline.h`](ai_stalker_inline.h.md) · [`ai_stalker.cpp`](ai_stalker.cpp.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](../../CharacterPhysicsSupport.cpp.md) · [`Helicopter2.cpp`](../../Helicopter2.cpp.md) · [`LevelGraphDebugRender.cpp`](../../LevelGraphDebugRender.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](../../Level_bullet_manager_firetrace.cpp.md) · [`agent_corpse_manager.cpp`](../../agent_corpse_manager.cpp.md) · [`agent_enemy_manager.cpp`](../../agent_enemy_manager.cpp.md) · [`agent_explosive_manager.cpp`](../../agent_explosive_manager.cpp.md) · [`agent_location_manager.cpp`](../../agent_location_manager.cpp.md) · [`agent_member_manager.cpp`](../../agent_member_manager.cpp.md) · [`ai_monsters_misc.cpp`](../ai_monsters_misc.cpp.md) · [`ai_stalker.cpp`](ai_stalker.cpp.md) · [`ai_stalker_cover.cpp`](ai_stalker_cover.cpp.md) · [`ai_stalker_debug.cpp`](ai_stalker_debug.cpp.md) · [`ai_stalker_events.cpp`](ai_stalker_events.cpp.md) · _and 110 more_
**Tier floor** — T2: a large mutable aggregate with owned sub-managers and per-frame caches; nothing here needs manual layout

## Purpose

Declares the class implemented across nine source files. A stalker is a human non-player
character: it plans, fights, takes cover, throws grenades, trades, talks, and can be
critically wounded rather than killed. It is the only creature in the game driven by the
goal/plan/action layer of chapter 14 rather than by a hand-written priority ladder.

The class is enormous because it is the *meeting point* of eight subsystems that each own a
slice of a human: the planner, the weapon-handling planner, the movement manager, the sight
manager, the animation manager, perception, the squad agent manager, and the inventory
owner. Almost every member here is either a handle to one of those, a cache the subsystems
share, or a tuned number read from configuration.

**The file split is by concern, not by size,** and a rebuilder should keep it:

| File | Carries |
|---|---|
| [`ai_stalker.cpp`](ai_stalker.cpp.md) | lifecycle, configuration, the update loop, the two brains |
| [`ai_stalker_fire.cpp`](ai_stalker_fire.cpp.md) | weapon choice, aim, friendly-fire avoidance, grenades, critical wounds |
| [`ai_stalker_cover.cpp`](ai_stalker_cover.cpp.md) | cover selection and its cache |
| [`ai_stalker_misc.cpp`](ai_stalker_misc.cpp.md) | what counts as an enemy or a useful item, squad reactions |
| [`ai_stalker_events.cpp`](ai_stalker_events.cpp.md) | inventory events and touch |
| [`ai_stalker_feel.cpp`](ai_stalker_feel.cpp.md) | perception relevance rules |
| [`ai_stalker_script.cpp`](ai_stalker_script.cpp.md) | the planner vocabulary exported to Lua |
| [`ai_stalker_script_entity.cpp`](ai_stalker_script_entity.cpp.md) | the Lua action-queue adapters |
| [`ai_stalker_debug.cpp`](ai_stalker_debug.cpp.md) | the diagnostic surface, debug builds only |

## State

Grouped by what owns it.

```text
RECORD Stalker

  # -- the four owned sub-managers ------------------------------------------
  animation  : StalkerAnimationManager    # torso/legs/head blending, script animations
  brain      : StalkerPlanner             # the goal/plan/action planner of chapter 14
  sight      : SightManager               # where the head and eyes point
  movement   : SmartCoverMovementManager  # paths, body state, and smart-cover occupancy

  # -- rank-derived multipliers, computed once at spawn ---------------------
  rank_immunity, rank_visibility, rank_dispersion : real

  # -- weapon dispersion, eight cases read from configuration ---------------
  dispersion : map<(gait, stance, zoomed), real>

  # -- the "best item to kill with" cache -----------------------------------
  item_cache_valid      : bool
  best_item_to_kill     : optional<Item>       # in my hands, or reachable
  best_item_value       : real
  best_ammo             : optional<Item>
  best_found_item_to_kill, best_found_ammo : optional<Item>   # remembered, not held

  # -- six cover evaluators, each with its own inertia ----------------------
  cover_evaluators : { close_to_enemy, far_from_enemy, best, angle, safe, ambush }

  # -- the cover cache ------------------------------------------------------
  best_cover        : optional<CoverPoint>
  best_cover_value  : real
  best_cover_actual : bool
  best_cover_advance_cover : optional<CoverPoint>
  cover_subscribers : list<callback(new_cover, old_cover)>

  # -- friendly-fire cache, valid for one frame -----------------------------
  can_kill_enemy, can_kill_member : bool
  pick_distance   : real
  pick_frame_id   : int

  # -- grenade throwing -----------------------------------------------------
  throw_valid      : bool
  throw_target, throw_position, throw_velocity, throw_collide_position : vector3
  computed_position, computed_direction : vector3   # what the cached throw was computed for
  throw_enabled    : bool
  throw_ignore     : optional<Object>
  last_throw_time  : int
  throw_interval   : int (milliseconds)
  can_throw        : bool

  # -- fire queue parameters: six weapon classes x three ranges x four numbers
  queue : map<(weapon_class, range_band), (min_size, max_size, min_interval, max_interval)>
  queue_fire_distance : map<weapon_class, (medium, far)>

  # -- state flags ----------------------------------------------------------
  wounded, group_behaviour, can_select_weapon, can_select_items : bool
  take_items_enabled, death_sound_enabled : bool
  sniper_update_rate, sniper_fire_mode : bool
  registered_in_combat_on_migration : bool

  critical_wound_weights : list<real>   # per body part, from the character record
  aim_bone_id            : text
  hit_callback           : optional<callback(hit) -> bool>
  ignored_touched_objects : list<Object>
  bone_protection        : optional<BoneProtectionTable>
```

**Invariants**

- Every cache here carries its own validity flag or frame stamp, and each is invalidated by
  a *different* event: the item cache by taking or dropping anything or by the enemy
  changing; the cover cache by the enemy changing, the restrictions changing, a danger
  location appearing or disappearing, or the cover being blocked; the friendly-fire cache
  by the frame advancing; the throw cache by the stalker moving or turning. Getting these
  invalidations right is most of what makes the stalker feel responsive rather than stuck.
- `best_item_to_kill` and `best_ammo` are the same object when the stalker holds a working
  weapon, and differ when it holds a weapon with no ammunition and knows where ammunition is.
- The grenade-throw cache is keyed on the stalker's own position *and* direction, because
  the launch point is derived from both.

## Exported units

The public surface runs to roughly two hundred entry points. They fall into eleven groups;
each group's contracts live in the file named above.

- **the cast battery and lifecycle** — construct, load, reload, spawn, destroy, save,
  restore, re-initialise.
- **the update loop** — the frame update, the scheduled update, and `Think`.
- **the two brains** — the goal/plan/action planner, and the separate weapon-handling
  planner inherited from the object handler.
- **perception policy** — what is worth looking at, what may be touched, what counts as an
  enemy, what counts as a useful item.
- **weapon handling** — accuracy, fire parameters, ammunition choice, the fire-queue
  parameter accessors (one per weapon class, range band and field — a hundred and eight of
  them), and the friendly-fire query.
- **cover** — find, cache, invalidate, subscribe.
- **grenades** — target, trajectory check, force, completion.
- **critical wounds** — the body-part weights, the entry and exit conditions, and the squad
  vocalisations they trigger.
- **squad participation** — the agent manager accessor, team migration, group behaviour.
- **trade and inventory** — item selection, sale rules, conflict resolution between weapons.
- **script adapters** — the Lua action queue, and the planner vocabulary.
- **diagnostics** — debug builds only.
