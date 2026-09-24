# src/xrGame/script_game_object_script3.cpp

> The second half of the game object's script surface: sounds, sight, restrictors, information portions, tasks, news, relations, anomalies, and the explicit downcast family.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`pda_space.h`](pda_space.h.md) · [`memory_space.h`](memory_space.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`GameTask.h`](GameTask.h.md) · [`script_ini_file.h`](../xrServerEntities/script_ini_file.h.md) · [`Car.h`](Car.h.md) · [`helicopter.h`](helicopter.h.md) · [`HangingLamp.h`](HangingLamp.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`ZoneCampfire.h`](ZoneCampfire.h.md) · [`PhysicObject.h`](PhysicObject.h.md) · [`Artefact.h`](Artefact.h.md) · [`level_changer.h`](level_changer.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_game_object_script.cpp`](script_game_object_script.cpp.md)
**Tier floor** — T2: registration data only

## Purpose

The other link in the registration chain assembled by
[`script_game_object_script.cpp`](script_game_object_script.cpp.md). Again the division is
arbitrary; three things in it are not.

## `script_register_game_object2`

**Contract** — adds three constant tables and the remaining method exports.

```text
game_object.EPdaMsg         = { dialog_pda_msg, info_pda_msg, no_pda_msg }
game_object.ACTOR_RELATIONS = { relation_attack, relation_fight_help_monster,
                                relation_fight_help_human, relation_kill }
game_object.CLSIDS          = { no_pda_msg }     # one entry; see the notes
```

## Collections returned as iterators

**Invariants** — the four memory collections — visible objects, sound objects, hit objects
and not-yet-visible objects — are exported as **iterators over the engine's own list**,
not as copied tables. Three consequences a rebuild must honour:

- iterating is cheap and does not allocate, which matters because a script may walk a
  creature's visible set every frame;
- the list is **live**: anything that changes the creature's memory during the walk
  invalidates it, and the engine does not detect that;
- a script cannot retain the collection past the loop.

A rebuild whose script layer cannot expose a borrowed sequence must copy, and must then
accept the per-frame allocation or change the scripts.

## Output parameters and ownership

**Invariants** —

- `accessible_nearest` writes its answer into an **output position** which the binding
  layer turns into a second return value. Scripts see two results: the vertex and the
  position. A rebuild returning a pair is equivalent and clearer.
- `give_task` **takes ownership** of the task object the script built. The task lives in the
  player's journal afterwards and the script must not keep a reference.

## Shorter spellings registered as adapters

**Invariants** — several methods are registered twice, once directly and once through a
small adapter supplying a default:

- `get_task_state` and `set_task_state` without an objective index address the task's
  **root objective**, which is what a single-step quest wants.
- `give_task` without a timer passes no timer.
- `give_game_news` has a form matching the **oldest of the three games'** argument shape,
  which took an image rectangle the current engine ignores. It is accepted and discarded so
  that scripts written for that game load unchanged.

## The downcast family

**Contract** — about thirty methods of the form `cast_<kind>` returning the object viewed
as that kind, or nothing.

**Notes**

These exist because the binding layer used here **casts to the most-derived type rather
than to the requested one**: asking the player for its weapon view returned the player, not
nothing, so a script could not test the result. The fix is an explicit conversion per kind,
each performing the downcast itself and answering nothing on failure.

This is the clearest incidental-to-essential translation in the whole facade. The
*mechanism* is a workaround for one script binding library; the *decision* is that a script
must be able to ask "is this object usable as a weapon" and get a usable handle or a clean
no. A rebuild whose script layer answers that correctly needs neither these nor the
[class predicates](script_game_object4.cpp.md#class-predicates) that duplicate them — but
must export both name sets as aliases, because shipped scripts use both.

A further dozen conversions are present but disabled, the same set as the disabled class
predicates, and for the same reason: the includes were not wanted. A rebuild provides the
whole set.

`CLSIDS` is a table with a single entry that duplicates a value from the message table and
has nothing to do with class identifiers. It is vestigial; a rebuild exports it only if
some shipped script reads it, and nothing in the source says one does.
