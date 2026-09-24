# src/xrGame/script_game_object.h

> Declares the game object — the single facade through which every script reaches every entity in the world.

**Needs** — [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`script_bind_macroses.h`](script_bind_macroses.h.md) · [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`game_location_selector.h`](game_location_selector.h.md) · [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`xr_time.h`](xr_time.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ActorVehicle.cpp`](ActorVehicle.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`CarScript.cpp`](CarScript.cpp.md) · [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md) · [`GameObject.cpp`](GameObject.cpp.md) · [`Helicopter2.cpp`](Helicopter2.cpp.md) · [`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md) · [`PhraseScript.cpp`](PhraseScript.cpp.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`action_base_script.cpp`](action_base_script.cpp.md) · [`action_planner_action_script.cpp`](action_planner_action_script.cpp.md) · [`action_planner_action_script_inline.h`](action_planner_action_script_inline.h.md) · [`action_planner_script.cpp`](action_planner_script.cpp.md) · [`action_script_base_inline.h`](action_script_base_inline.h.md) · _and 69 more_
**Tier floor** — T2: a declaration, but an enormous one

## Purpose

Declares the **game object**: one handle, held one-per-client-object, exposing everything
the script layer is allowed to touch on *any* entity — actor, creature, weapon, artefact,
anomaly, vehicle, lamp, container, door. It is the widest frozen interface in the
repository (conformance criterion 10), and its size is the design: rather than exporting a
class hierarchy, the engine exports one flat type whose methods each downcast internally
and complain if the entity is the wrong kind. The mechanism behind that is in
[`script_game_object_impl.h`](script_game_object_impl.h.md) and
[`script_bind_macroses.h`](script_bind_macroses.h.md); the contracts are spread across nine
implementation files, listed below.

One design consequence is worth stating before the list: **the facade has almost no state
and no behaviour of its own**. It holds a pointer to a client object and a door
registration, and every method is a guarded delegation. A rebuild may generate this entire
file.

## State

```text
RECORD GameObject
  client_object : ClientObject    # the live instance this facade fronts; never changes
  door          : optional<Door>  # a registration in the navigation layer's door table,
                                  # owned here because scripts, not the engine, decide
                                  # which physics objects count as doors
```

**Invariant** — a facade and its client object point at each other. The facade is destroyed
with the object, and using one afterwards is the single most common script bug; the debug
build catches it in
[`script_game_object_impl.h`](script_game_object_impl.h.md).

## Where the implementations live

| File | Covers |
|---|---|
| [`script_game_object_use.cpp`](script_game_object_use.cpp.md) | construction, destruction, identity, kill, relations, callbacks, physics scripting |
| [`script_game_object.cpp`](script_game_object.cpp.md) | transform, condition, action queue, weapon ammo, inventory basics, actor-menu entry points |
| [`script_game_object2.cpp`](script_game_object2.cpp.md) | memory, explosives, item handling, actor placement, stalker thresholds |
| [`script_game_object3.cpp`](script_game_object3.cpp.md) | cover, movement parameters, sight, trade tuning, anomalies, upgrades, bones |
| [`script_game_object4.cpp`](script_game_object4.cpp.md) | sound player, wounds, inventory boxes, particles, class predicates |
| [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) | information portions, dialogue, tasks, reputation, inventory, restrictors, doors, weight |
| [`script_game_object_smart_covers.cpp`](script_game_object_smart_covers.cpp.md) | smart covers and loopholes |
| [`script_game_object_trader.cpp`](script_game_object_trader.cpp.md) | trader animation and speech |
| [`script_game_object_use2.cpp`](script_game_object_use2.cpp.md) | per-species monster controls |

## Exported units, by subject

**Identity and transform** — `parent`, `class_id`, `name`, `section`, `position`,
`direction`, `centre`, `mass`, `id`, `story_id`, `visible`, `enabled`, `spawn_ini`,
`visual_name`, `spatial_type`, `level_vertex_id`, `game_vertex_id`.

**Condition and life** — `alive`, `kill`, `death_time`, `max_health`, `accuracy`, `team`,
`squad`, `group`, and the seven condition axes (health, psi-health, power, radiation,
satiety, bleeding, morale) each with a read and a *delta* write; `set_health_ex` writes
health absolutely, which is why it exists separately.

**Damage** — `hit` applies an authored hit; `who_hit_name` and `who_hit_section_name` name
the last attacker; `reset_bone_protections` re-reads a creature's per-bone armour.

**Script control** — `set_script_control`, `get_script_control`, `get_script_control_name`,
`can_script_capture`, `add_action`, `current_action`, `action_count`, `action_by_index`,
`reset_action_queue`, `patrol_path_name`, `binded_object`, `bind_object`.

**Senses and memory** — `memory_visible_objects`, `memory_sound_objects`,
`memory_hit_objects`, `not_yet_visible_objects`, `memory_time`, `memory_position`,
`enable_memory_object`, `enable_vision`, `vision_enabled`, `visibility_threshold`,
`set_sound_threshold`, `restore_sound_threshold`, `best_enemy`, `best_danger`, `best_item`,
`check_object_visibility`, `check_type_visibility`, the four `remove_*` memory pruners,
`make_object_visible_somewhen`, `set_enemy_callback`.

**Movement** — body state, movement type, mental state, path type and detail path type,
each with a setter, a current reader and a target reader; destination by level vertex, game
vertex, position or patrol path; `path_completed`, `movement_target_reached`,
`enable_movement`, `extrapolate_length`, `location_on_path`, `vertex_in_direction`,
`is_body_turning`, `head_orientation`, the three `inactualize_*` path invalidators,
`set_patrol_extrapolate_callback`.

**Restrictors** — `add_restrictions`, `remove_restrictions`, `remove_all_restrictions`, the
four restriction readers, `accessible_position`, `accessible_vertex_id`,
`accessible_nearest`, `restriction_type`.

**Cover and smart cover** — `best_cover`, `safe_cover`, `find_best_cover`, and the
smart-cover block (see
[`script_game_object_smart_covers.cpp`](script_game_object_smart_covers.cpp.md)).

**Sight** — nine `set_sight` overloads and `sight_params`.

**Sound** — `add_sound`, `add_combat_sound`, `remove_sound`, `set_sound_mask`, six
`play_sound` arities, `active_sound_count`, `sound_prefix`, `sound_voice_prefix`.

**Animation** — `play_cycle`, `add_animation`, `clear_animations`, `animation_count`,
`animation_slot`, `play_hud_motion`, `switch_state`, `state`, `bone_position`, `bone_id`,
`bone_visible`, `set_bone_visible`, `start_particles`, `stop_particles`.

**Inventory** — item lookup by name, index and slot; active item and active slot;
`iterate_inventory`, `for_each_inventory_item`, `drop_item`, `drop_item_and_teleport`,
`transfer_item`, `make_item_active`, `eat`, `belt` queries, `mark_item_dropped`,
`unload_magazine`, weight and carrying-capacity accessors.

**Items** — condition and cost; weapon addon state and attach/detach; ammunition type and
count; ammunition-box contents; grenade mode; `remaining_uses`; upgrade add/install/query
and iteration; artefact restoration rates.

**Social** — information portions (`give`, `disable`, `has`, `time`), goodwill in four
forms, relation, sympathy, community goodwill, attitude, rank, reputation, community,
profile, character name and icon.

**Dialogue, trade and tasks** — talk enable/stop/query, trade and upgrade enable/query,
`run_talk_dialog`, `start_dialog` family, `switch_to_trade` / `switch_to_upgrade`,
`give_task_to_actor`, `task`, `active_task`, `game_task_state`, game news, iconed talk
messages.

**Zones, containers, vehicles, lamps** — anomaly enable/disable/power, inventory-box
closed/take state and emptiness, `get_car`, `get_helicopter`, `get_hanging_lamp`,
`get_custom_holder`, `get_current_holder`, `attach_vehicle`, `detach_vehicle`,
`get_campfire`, `get_artefact`, `get_physics_object`, `get_physics_shell`.

**Doors** — `register_door`, `unregister_door`, `on_door_is_open`, `on_door_is_closed`,
`lock_door_for_npc`, `unlock_door_for_npc`, `is_door_locked_for_npc`,
`is_door_blocked_by_npc`.

**Callbacks** — `set_callback` (three arities) and `clear_callbacks`, keyed by a callback
type enumeration; `set_fastcall` and `set_const_force` schedule work in the physics loop.

**Class predicates** — a flat family of `is_*` tests (`is_actor`, `is_weapon`,
`is_stalker`, `is_monster`, `is_anomaly`, `is_artefact`, `is_ammo`, …) and, separately, a
family of `cast_*` accessors. The predicates answer a boolean; the casts return a typed
handle or nothing.

**Free functions** — `sell_condition`, `buy_condition`, `show_condition` in both a
configuration-section form and a two-factor form, operating on the *default* trade
parameters shared by every trader who does not override them.
