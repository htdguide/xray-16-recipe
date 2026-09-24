# src/xrGame/script_game_object_script2.cpp

> The first half of the game object's script surface: the condition properties, identity, the action queue, the memory and movement vocabularies, and the smart-cover methods.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_entity_space.h`](script_entity_space.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`memory_space.h`](memory_space.h.md) · [`cover_point.h`](cover_point.h.md) · [`script_hit.h`](script_hit.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`script_binder_object.h`](script_binder_object.h.md) · [`script_sound_info.h`](script_sound_info.h.md) · [`script_monster_hit_info.h`](script_monster_hit_info.h.md) · [`action_planner.h`](action_planner.h.md) · [`danger_object.h`](danger_object.h.md) · [`smart_cover_object.h`](smart_cover_object.h.md) · [`relation_registry.h`](relation_registry.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_game_object_script.cpp`](script_game_object_script.cpp.md)
**Tier floor** — T2: registration data only

## Purpose

One link in the registration chain assembled by
[`script_game_object_script.cpp`](script_game_object_script.cpp.md). It takes the partly
built declaration, adds its share, and returns it. The share is arbitrary; what is worth
recording is the handful of registrations that are not plain method exports.

## `script_register_game_object1`

**Contract** — adds four constant tables, the seven condition **properties**, and several
hundred method exports.

```text
game_object.relation     = { friend, neutral, enemy, dummy }
game_object.action_types = { movement, watch, animation, sound, particle, object,
                             action_type_count }
game_object.EPathType    = { game_path, level_path, patrol_path, no_path }
game_object.ESelectionType = { alifeMovementTypeMask, alifeMovementTypeRandom }
```

**Invariants**

- The seven condition axes — health, psi-health, power, satiety, radiation, morale,
  bleeding — are exported as **properties**, read and written with field syntax, while
  everything else on the facade is a method. That is why the delta-writing trap is so easy
  to fall into: `obj.health = 0.1` *adds* ten percent. See
  [`script_game_object.cpp`](script_game_object.cpp.md#condition-axes).
- `bleeding` is exported both as a property and as a method, because the property spelling
  arrived later and the method spelling is in shipped scripts.
- `see` is **one name over two different questions**: given an object it asks whether this
  creature can see that object; given text it asks whether this creature can see *any*
  object of that configuration section. Both ship.
- `action` — the current action of the queue — is registered with an **ownership transfer**:
  the copy it returns becomes the script's to release. This is the only place the facade
  hands out ownership, and the copy exists so that a script cannot reach into a running
  action; see
  [`script_game_object.cpp`](script_game_object.cpp.md#action-queue).
- `bind_object` likewise **takes ownership** of the binder the script hands it: the object
  keeps it for its lifetime, and the script must not keep a reference.
- The door methods are exported under names ending "for npc" —
  `register_door_for_npc`, `lock_door_for_npc` and so on — while the facade's own methods
  are not so named. The script spelling is the frozen one and it says what the table is
  for: these doors exist for *creature pathing*, not for the player.

**Notes**

The path-type table's two selection modes are named with an `alife` prefix that has nothing
to do with the alife simulation; they select how a creature picks its next cross-level
branch. The names are frozen and misleading, and a rebuild should keep them and say so.

`action_type_count` is exported as if it were an action type. It is the count, used by
scripts as a bound.
