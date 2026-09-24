# src/xrGame/script_game_object.cpp

> The first slice of the game object facade: transform, condition, the action queue, weapon ammunition, and the entry points that open the actor's menu.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`script_bind_macroses.h`](script_bind_macroses.h.md) · [`script_entity.h`](script_entity.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomMonster.h`](CustomMonster.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`space_restrictor.h`](space_restrictor.h.md) · [`Actor.h`](Actor.h.md) · [`InventoryBox.h`](InventoryBox.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`script_game_object.h`](script_game_object.h.md)
**Tier floor** — T2: guarded delegation, plus one bone-transform composition

## Purpose

The facade is split across nine files for compile-time reasons only; the split lines are
arbitrary and a rebuild should ignore them. This file carries the most-used third of the
surface. What it *does* establish, in its first fifty lines, is the pattern every one of
those nine files follows.

## State

`Stateless.` The facade's two fields are declared in
[`script_game_object.h`](script_game_object.h.md) and constructed in
[`script_game_object_use.cpp`](script_game_object_use.cpp.md).

## The guarded-delegation pattern

Every method on the facade is one of these shapes, and the shapes are generated from a
small set of templates in [`script_bind_macroses.h`](script_bind_macroses.h.md):

```text
FUNCTION accessor() -> T
  target = view the client object as <concrete class>
  IF target is none THEN
    log script error "<concrete class> : cannot access class member <name>"
    RETURN <a named fallback value>
  RETURN target.<expression>

FUNCTION mutator(args)
  target = view the client object as <concrete class>
  IF target is none THEN
    log script error "<concrete class> : cannot access class member <name>"
    RETURN
  target.<expression>(args)
```

**Invariants**

- The fallback value is declared *per method*, not per type, and it is part of the frozen
  contract: scripts test for it. The engine's conventions are `-1` for a numeric quantity,
  the zero vector for a position, empty text for a name, `false` for a predicate, and
  nothing for a handle.
- A failed downcast is a *script error*, never a fault. Shipped scripts call weapon methods
  on non-weapons routinely, and the game must keep running.
- A handful of methods abort instead of returning a fallback, because no fallback is
  meaningful: those returning a reference to a collection, and `get_car` and
  `get_helicopter`, which scripts call only after testing the class.

**Notes**

Three escalation levels appear across the nine files and the choice between them is
load-bearing, because it decides whether a modder's mistake is a log line or a crash:
*log and fall back* (the overwhelming majority), *assert* (used where the caller is engine
code, e.g. the anomaly and artefact accessors), and *silently do nothing* (used where the
call is expected to be speculative, e.g. `phantom_set_enemy`).

## Transform and identity

**Contract** — `position`, `direction`, `centre`, `mass`, `id`, `visible`, `enabled` and
`story_id` each read one field of the client object through the pattern above, with the
class requirement as loose as possible: only `mass` needs a physics-capable object.
`level_vertex_id` and `game_vertex_id` read the entity's recorded navigation position —
both graphs, because an entity's location is a graph vertex, not a coordinate.
`spawn_ini` returns the entity's own authored configuration fragment, the per-instance
override of its section.

## Condition axes

**Contract** — seven quantities (health, psi-health, power, satiety, radiation, bleeding,
morale) readable on any living entity, plus the field-of-view and range of its vision.
Every *write* is a delta, not an assignment: `set_health(0.1)` adds ten percent. That is a
frozen trap and the reason
[`set_health_ex`](script_game_object4.cpp.md) exists alongside it.

## Action queue

**Contract** — `add_action`, `current_action`, `action_by_index` and `reset_action_queue`
forward to the [action queue mixin](script_entity.cpp.md) on the entity, failing with a
script error on an entity that has no such mixin.

`current_action` **allocates a copy** and hands it to the script, which then owns it. The
queue's own action must not escape, because the script would otherwise be able to mutate a
running action's completion flags.

## Bone access

**Contract** — `bone_id` resolves a bone name on the entity's skeleton; `bone_position`
returns a named bone's world position, or the skeleton root's when given an empty name.

```text
FUNCTION bone_position(bone_name) -> vector
  bone = IF bone_name is empty THEN skeleton root ELSE lookup(bone_name)
  RETURN translation of (entity transform composed with bone's current pose)
```

## Weapon ammunition

**Contract** — `ammo_elapsed` and `set_ammo_elapsed` read and write the rounds in the
magazine; `suitable_ammo_total` counts every round in the owner's inventory that this
weapon can chamber; `ammo_type`, `set_ammo_type`, `has_ammo_type` and `ammo_count` address
the weapon's *list* of accepted ammunition by index into that list, so an out-of-range
index answers "none" rather than failing; `weapon_substate` exposes the firing state
machine's sub-state; `set_queue_size` limits burst length on a magazined weapon.

The fallback for the type-valued readers is 255, not −1, because they are byte-wide.

## Inventory item basics

**Contract** — `cost` and `condition` read an item's trade value and wear. `set_condition`
takes an *absolute* target and converts it to the delta the item's own interface wants,
which is the opposite convention from the condition axes above and is frozen that way.
`eat` makes an inventory owner consume one of its items.

## Restrictor containment

**Contract** — `inside(position)` and `inside(position, epsilon)` ask a restrictor volume
whether a point, or a small sphere around it, lies within. The one-argument form uses the
engine's smallest meaningful length as the radius, so the two forms differ only in
tolerance.

## Actor menu entry points

**Contract** — `use`, `start_trade` and `start_upgrade` open the actor's inventory screen
against this object as the partner. `use` first performs the entity's own use behaviour and
only then, if the *user* was the actor, opens a screen: the corpse-and-container screen for
an inventory box or an inventory owner. `start_trade` and `start_upgrade` skip the use
behaviour and open the trade and upgrade screens directly. All three do nothing when the
user is not the actor, since these screens exist only for the player.

**Notes**

`use` opening the corpse-search screen for a *living* inventory owner is deliberate in this
build — the original ran a talk dialogue instead, and that path is disabled. The behaviour
difference is visible in the game.

## Vehicle, lamp and holder accessors

**Contract** — `get_helicopter` and `get_hanging_lamp` abort when the object is not of that
type; `get_custom_holder` logs and returns nothing. The inconsistency is historical and a
rebuild should pick one.

## Movement tuning on a creature

**Contract** — `extrapolate_length` reads and writes how far ahead of the creature its
detail path is smoothed; `set_fov` and `set_range` retune its vision cone;
`vertex_in_direction` walks the navigation mesh from a vertex toward a direction and
returns the farthest vertex reached.

```text
FUNCTION vertex_in_direction(from_vertex, direction, max_distance) -> vertex
  IF the creature's restrictors forbid from_vertex THEN log error; RETURN none
  step = normalize(direction) * max_distance
  temporarily widen the creature's restrictors by max_distance around from_vertex
  result = walk from from_vertex toward (position(from_vertex) + step),
           stopping at the last vertex still on the mesh
  restore the restrictors
  RETURN result IF valid ELSE from_vertex          # never answer "nowhere"
```

The temporary widening is the interesting part: without it the walk would stop at the
restrictor boundary, and scripts use this call precisely to ask where a creature *could*
go.

## Miscellaneous single-purpose accessors

**Contract** — `invulnerable` reads and writes a creature's damage immunity;
`smart_cover_description` names the authored behaviour table a smart cover uses;
`phantom_set_enemy` retargets a phantom, silently on anything else; `physics_shell`
returns the object's rigid body or nothing; `set_patrol_extrapolate_callback` installs,
replaces or clears a script predicate consulted when a patrolling creature reaches the end
of its path.
