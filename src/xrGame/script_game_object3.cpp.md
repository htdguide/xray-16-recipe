# src/xrGame/script_game_object3.cpp

> Cover selection, the creature movement and sight surfaces, scripted animations, trade tuning, anomalies, artefacts, held-item state, bones and restrictors — the widest of the nine facade slices.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`memory_space.h`](memory_space.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`game_path_manager.h`](game_path_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`eatable_item.h`](eatable_item.h.md) · [`Artefact.h`](Artefact.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`Actor.h`](Actor.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`space_restrictor.h`](space_restrictor.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation, plus two cover queries and a set of validity gates

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across,
and the largest. The split line is arbitrary — see the facade's own twin — but the file
does contain the whole of two subjects a rebuild must get right: how a script commands a
creature's **movement**, and how it commands its **sight**. Both are *destination*
surfaces, not commands: a script writes what it wants and the creature's managers converge
on it over the following frames.

Everything follows the
[guarded-delegation pattern](script_game_object.cpp.md#the-guarded-delegation-pattern).
The sections below cover only what that pattern does not already say.

## State

`Stateless.`

## Cover selection

**Contract** — two queries against the level's precomputed cover data, both answering a
cover point or nothing:

```text
FUNCTION best_cover(from, enemy_position, radius, min_enemy_distance, max_enemy_distance)
  # "where near `from` can I stand that is hidden from `enemy_position`,
  #  and between min and max away from it"
  configure this stalker's own best-cover evaluator with the enemy and the distance band
  RETURN the cover manager's best point within radius of `from`, judged by that evaluator

FUNCTION safe_cover(from, radius, min_distance)
  # "where near `from` is exposed from as few directions as possible"
  configure this stalker's own safe-cover evaluator with the minimum distance
  RETURN the cover manager's best point within radius of `from`, judged by that evaluator
```

**Invariants**

- The evaluators are **owned by the stalker**, one of each, and are reconfigured in place
  on every call. Two scripts querying cover for one stalker in the same frame are safe only
  because each call configures immediately before it searches — a rebuild that caches the
  configuration across calls breaks that.
- The distance band on the best-cover query is why a creature does not retreat to the far
  end of the level: cover is wanted at a *fighting* distance, not merely out of sight.
- Both return nothing when no point qualifies, and the caller must handle it. There is no
  fallback "least bad" answer, because standing in poor cover reads worse than not moving.

## Movement destinations

**Contract** — a script sets a creature's movement **destination and manner** as
independent fields, each with a setter and most with both a current and a target reader:

```text
destination : set_dest_level_vertex_id(v), set_dest_game_vertex_id(v),
              set_desired_position(p) / set_desired_position()  — clear,
              set_desired_direction(d) / set_desired_direction() — clear,
              set_patrol_path(...), set_movement_selection_type(kind)
manner      : set_body_state, set_movement_type, set_mental_state,
              set_path_type, set_detail_path_type
readers     : body_state / target_body_state, movement_type / target_movement_type,
              mental_state / target_mental_state, path_type, detail_path_type,
              path_completed, head_orientation
invalidate  : inactualize_patrol_path, inactualize_level_path, inactualize_game_path,
              patrol_path_make_inactual
```

**Invariants**

- Every manner field has a **current** and a **target** reader, and they differ while the
  creature transitions. A script that reads the current value right after writing the
  target sees the old one — this is the single most common misunderstanding of the movement
  surface, and it is correct behaviour: standing up from a crouch takes an animation.
- **A destination vertex is validated twice** before it is accepted, and both rejections
  are silent refusals rather than failures:

```text
FUNCTION set_dest_level_vertex_id(vertex)
  IF vertex is not a real vertex of the level graph THEN
    report (debug builds only) naming the planner action that asked
    RETURN                                    # refuse
  IF this creature's restrictors forbid the vertex THEN
    report, naming the creature and both its permitted and forbidden restrictor sets
    RETURN                                    # refuse
  set the movement destination
```

  The restrictor check is the load-bearing half: a creature sent outside its permitted
  region would path to the boundary and stall there forever, with nothing in the log. The
  message names both restrictor sets precisely because that is the information needed to
  fix the script. The cross-level destination is validated for existence only — restrictors
  are a level concept.

- **A desired direction is normalized for the script**, and the original's handling of that
  differs per game: the direction is rejected outright if it is zero, reported if it is not
  unit length, and then normalized — except when running the oldest of the three games,
  where it is passed through untouched. A rebuild must keep the per-game split, because the
  old game's content passes unnormalized directions and relies on the resulting behaviour.
- The three invalidation calls exist because the path managers cache. A script that moves a
  creature, changes its restrictors, or re-authors its patrol must say so; nothing detects
  it.
- Clearing is again spelled as the same method with no argument.

## Sight

**Contract** — where a creature looks, written as a *sight action* with several shapes:

```text
set_sight(kind, direction)                       # look along a direction
set_sight(kind, direction, look_over_delay)      # ... after a delay
set_sight(kind, torso_look, follow_path)         # a kind that needs no target
set_sight(object)                                # track an entity
set_sight(object, torso_look)
set_sight(object, torso_look, fire_at_it)
set_sight(object, torso_look, fire_at_it, no_pitch)
set_sight(memory_entry, torso_look)              # look where it remembers something
```

**Invariants**

- **`torso_look` decides whether the body turns with the head.** A creature that tracks a
  target with its head alone keeps walking where it was going; one that turns its torso
  commits. That flag is the difference between glancing and engaging, and it is exposed on
  almost every shape for that reason.
- `fire_at_it` makes the sight target the *aim* target as well, which is how a creature
  shoots where it looks. Separating them lets a creature look at one thing and keep its
  weapon on another.
- `no_pitch` clamps the look to the horizontal, for targets above or below that the
  creature's skeleton cannot follow without breaking.
- Looking at a **memory entry** rather than an object is what makes a creature watch the
  place it last saw something. This is the sight surface's tie to the memory model, and it
  is how searching behaviour is scripted.
- A directional sight is checked for unit length and normalized, with the same per-game
  exception as the movement direction — and here it is two games that are exempt, not one.
  The tolerance is a hundredth.

`head_orientation` reads back the creature's current head aim as a direction, negating both
angles to convert from the engine's internal rotation convention. Its failure value is the
largest representable coordinates, not the zero vector.

## Scripted animations

**Contract** — `add_animation(name, uses_hands, uses_movement_controller)` and
`add_animation(name, uses_hands, position, rotation, is_local)` queue an animation on a
stalker; `clear_animations` empties the queue; `animation_count` reports its depth.

**Invariants**

- Refused, with a report, when a **global animation selector** is installed — something
  else already owns the creature's animation and a script animation would fight it.
- **Reported but not refused** when the creature is in a smart cover. The report fires and
  the animation is queued anyway, which is almost certainly a missing early return: the
  smart-cover animation system will then contend with the queued one. A rebuild should
  refuse.
- `uses_hands` tells the animation system whether the creature's weapon must be put away
  first, which changes how long the animation takes to start.
- The placed form takes a position and rotation the animation plays *at*, with a flag
  saying whether those are relative to the creature or absolute in the world. Scripted
  set-pieces need the absolute form to line a creature up with authored scenery.

## Trade tuning

**Contract** — sell, buy and show conditions, each settable either from a named section of
a configuration file or from a friend factor and an enemy factor directly, per inventory
owner *and* as an engine-wide default (the default-setting forms are free functions rather
than facade methods). Plus `buy_item_condition_factor`, which scales an item's price by its
wear, and `buy_supplies`, which restocks a trader from an authored section.

**Invariants** — the two factors are the price multipliers applied to a *friend* and to an
*enemy*, with the creature's actual relation interpolating between them. That is the whole
trade model as far as scripts are concerned: reputation moves a price along one line
between two authored endpoints.

## Anomalies

**Contract** — `enable_anomaly`, `disable_anomaly`, `get_anomaly_power` and
`set_anomaly_power` on a zone.

**Invariants** — all four **abort** on a non-zone rather than logging. This is the facade's
documented assert level, used where the caller is expected to be engine-adjacent rather
than a modder's script.

## Artefact restore rates

**Contract** — five read/write pairs, one per condition axis an artefact acts on: health,
radiation, satiety, power and bleeding. Each is a *rate*, applied continuously while the
artefact is carried, not a one-off amount.

**Notes**

There are five here and seven condition axes on the facade's condition surface. Psi-health
and morale have no artefact rate — no shipped artefact affects them — and a rebuild
extending the set should extend both sides together.

## Held-item state

**Contract** — `play_hud_motion(name, blend_in, state)`, `switch_state(state)` and
`get_state` drive the first-person animation and state machine of the item in the player's
hands. All three try the weapon interface first and fall back to the general held-item
interface. `play_hud_motion` answers zero when no animation by that name exists on the
item, rather than failing; `get_state` answers the all-ones marker when the object is not a
held item at all. `weapon_in_grenade_mode` asks an under-barrel weapon which barrel is
selected.

## Bones, light and placement

**Contract** —

- `set_bone_visible(name, visible, recursive)` / `is_bone_visible(name)` — show or hide a
  part of a skinned model, optionally with its children. The write is skipped when the bone
  is already in the requested state, because toggling re-evaluates the whole subtree.
- `luminocity` and `luminocity_hemi` — how brightly lit the object is, directly and from
  the sky, as the renderer computed it. These are what a stealth script reads to decide
  whether the player is visible.
- `force_set_position(position, activate_physics)` — teleport an object that has a rigid
  body, writing the transform into **both** the rigid body and the character controller if
  it has one, after optionally waking the body. Missing either leaves the two
  representations disagreeing, which shows as an object snapping back on the next step.
- `set_remaining_uses` / `get_remaining_uses` / `get_max_uses` — the consumable-charge
  count on an edible item; all silently answer zero on anything else.

## Restrictors, spatial type, and touch iteration

**Contract** —

- `get_restriction_type` / `set_restriction_type` — read and write which kind of restrictor
  a volume is. **Setting a non-none type also registers the volume with the level's
  restriction manager**, which is what makes the change take effect; the register is not a
  separate call and a rebuild must not split them.
- `set_spatial_type` / `get_spatial_type` — the bits saying which of the engine's spatial
  indexes an object belongs to. Writing these directly re-classifies a live object, which
  is powerful and unguarded.
- `iterate_feel_touch(function)` — calls a script function once per object currently
  touching this one, passing each object's identifier. It passes the *identifier*, not the
  facade, so a script must look each one up; that keeps the iteration cheap and stops a
  script retaining facades of objects that may be about to leave.

## The rest

Enemy, corpse, current weapon, current outfit and its per-damage-kind protection, food and
medical item selection from an inventory, inventory counting and lookup by name or index,
team and squad assignment, visual-memory enabling, type-visibility checks against a
configuration section, dead-body and weapon-selection permissions, rank, sound prefix,
ammunition counts and box size, vehicle attach and detach, the patrol path name, and
`jump(position, factor)` for a monster's scripted leap. All are single guarded delegations;
their contracts are what their names say.

**Notes**

`get_patrol_path_name` is the file's one two-stage downcast: it tries the stalker interface
first and the [scripted-entity mixin](script_entity.h.md) second, because both kinds of
entity walk patrol paths by different machinery. It is the only method in the facade that
tries a second interface before giving up, and it is what lets one script drive a patrol on
either.

`set_movement_selection_type` reports a wrong-type object and then **dereferences it
anyway**. A rebuild must add the missing early return; this is a fault waiting on a script
error.
