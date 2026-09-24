# src/xrGame/visual_memory_manager.h

> Declares a creature's sight: the accumulator that decides when an object has been *noticed*, and the set of sightings it goes on believing in afterwards.

**Needs** — [`visual_memory_manager.cpp`](visual_memory_manager.cpp.md) · [`visual_memory_params.h`](visual_memory_params.h.md) · [`memory_space.h`](memory_space.h.md) · [`memory_space_impl.h`](memory_space_impl.h.md) · [`visual_memory_manager_inline.h`](visual_memory_manager_inline.h.md)
**Used by** — [`CameraLook.cpp`](CameraLook.cpp.md) · [`CarInput.cpp`](CarInput.cpp.md) · [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md) · [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`bloodsucker_predator_inline.h`](ai/monsters/bloodsucker/bloodsucker_predator_inline.h.md) · [`control_path_builder.cpp`](ai/monsters/control_path_builder.cpp.md) · [`monster_corpse_memory.cpp`](ai/monsters/monster_corpse_memory.cpp.md) · [`monster_enemy_manager.cpp`](ai/monsters/monster_enemy_manager.cpp.md) · [`monster_enemy_memory.cpp`](ai/monsters/monster_enemy_memory.cpp.md) · [`psy_dog_aura.cpp`](ai/monsters/pseudodog/psy_dog_aura.cpp.md) · [`ai_stalker_feel.cpp`](ai/stalker/ai_stalker_feel.cpp.md) · _and 23 more_
**Tier floor** — T2: small record vectors touched every scheduled update, plus a frozen save format

## Purpose

Declares the surface implemented in
[`visual_memory_manager.cpp`](visual_memory_manager.cpp.md). This is the vision half of the
three memory managers a creature owns — vision, sound, hit — and the only one that is
*polled*: sound and hit arrive as `feel` events, sight is recomputed from the frustum every
scheduled update.

The declaration carries one decision worth stating here rather than in the implementation:
**three different kinds of owner construct the same manager.** A monster, a stalker, and a
standalone `vision_client` (a sensor with a camera and no body) each get their own
constructor, and exactly one of the three owner references is set. Everything downstream
branches on which. A rebuild should express this as one manager parameterized by an *eye
source* — something that yields a position, a direction, a field of view and a range —
rather than as three owner pointers, because that is the only thing the three cases differ
in that the algorithm cares about.

Exported units:

- `update(time_delta)` — one sensing pass: refresh the frustum candidates, age the
  accumulators, expire stale sightings, evict.
- `visible(object, time_delta)` — the accumulator step; true once the object is noticed.
- `visible(level vertex, yaw, field of view)` — the navigation-position variant, answered
  by the level graph rather than by a ray.
- `visible_right_now` / `visible_now` — seen this pass, versus still believed in.
- `add_visible_object` — record a sighting, optionally *fictitious* (shared from a squad
  mate or planted by a script).
- `visible_object` / `visible_object_time_last_seen` / `not_yet_visible_object` — lookups.
- `feel_vision_mtl_transp(object, element)` — how much sight a struck surface lets
  through; the callback the `feel` layer uses mid-ray.
- `reload(section)` / `reinit` — configuration and lifecycle reset.
- `save` / `load` / `on_requested_spawn` — persistence, with the deferred path for a
  sighting of an entity that has not spawned yet.
- `remove` / `remove_links` — forget one record, or every record naming an object.
- `enable(object, flag)` / `enable(flag)` / `enabled` — suppress one sighting, or stop
  sensing without forgetting.
- `set_squad_objects` — aim this creature's sighting list at a shared one.
- `objects` / `raw_objects` / `not_yet_visible_objects` — read the three sets.
- `current_state` — which of the two tuned profiles is live.
- `mask` — this owner's bit in the squad's visibility bitset.
- `visibility_threshold` / `transparency_threshold` — the live profile's two thresholds.

## State

See [`visual_memory_manager.cpp`](visual_memory_manager.cpp.md) for the records and the
invariants. Two shapes are declared only here:

```text
RECORD DelayedVisibleObject          # a loaded sighting waiting for its subject to spawn
  object_id      : int (16-bit)      # entity identifier, as written in the save
  visible_object : VisibleObject     # fully formed except for the object reference

# the three parallel sets the manager keeps
raw_visibles  : list<ref game object>   # this pass's geometric candidates; scratch
visibles      : list<VisibleObject>     # the remembered sightings; borrowed, see .cpp
not_yet       : list<NotYetVisibleObject>  # accumulators for candidates below threshold
```

**Notes** — The remembered list is declared as a *pointer* to a vector rather than a vector,
and that indirection is the whole squad-sharing mechanism rather than a storage
optimization: absent means "this owner is dead or not yet registered", and shared means
"one member's sighting is the squad's sighting". See
[`visual_memory_manager_inline.h`](visual_memory_manager_inline.h.md) for the rule that
clearing it also clears the accumulators.
