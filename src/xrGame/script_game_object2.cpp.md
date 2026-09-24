# src/xrGame/script_game_object2.cpp

> Explosives, the item-handling goal, animation cycles, applying a script-built hit, the creature memory surface, actor placement, and the thresholds that decide which enemies a stalker bothers with.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`script_hit.h`](script_hit.h.md) · [`script_zone.h`](script_zone.h.md) · [`Explosive.h`](Explosive.h.md) · [`object_handler.h`](object_handler.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`memory_space.h`](memory_space.h.md) · [`AI_PhraseDialogManager.h`](AI_PhraseDialogManager.h.md) · [`Actor.h`](Actor.h.md) · [`Car.h`](Car.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation, plus one network-message construction

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across.
Everything follows the
[guarded-delegation pattern](script_game_object.cpp.md#the-guarded-delegation-pattern);
two things here are not delegation at all and carry the file — applying a script-built hit,
which is the only place a script constructs a network event directly, and the creature
memory surface, which is the largest coherent subject in the facade.

## State

`Stateless.`

## `hit` — applying a script-built hit

**Contract** — takes a [script hit](script_hit.h.md), turns it into the engine's damage
event, and **sends it as a network message to this object** rather than calling the damage
path directly. Fails hard if the hit names no attacker.

```text
FUNCTION hit(script_hit)
  REQUIRE script_hit.draftsman is present  ELSE FAIL WITH "where is hit initiator"
  event.kind        = damage event addressed to this object's id
  event.attacker    = script_hit.draftsman's id
  event.weapon      = none                     # a scripted hit has no weapon
  event.direction   = script_hit.direction
  event.power       = script_hit.power
  event.bone        = resolve script_hit.bone_name on this object's skeleton,
                      or the root bone when the name is empty
  event.point_in_bone_space = origin           # always: a scripted hit has no impact point
  event.impulse     = script_hit.impulse
  event.type        = script_hit.type
  send event to this object
```

**Invariants**

- The hit goes **through the network event path even in single player**. That is the whole
  point: damage is authoritative state, so it must take the same route as a hit from a
  remote client, arrive in the same order relative to other events, and be recorded the
  same way. A rebuild that shortcuts to the damage function produces different ordering
  under load and a different save.
- A hit with no attacker **fails hard**, uniquely in the facade. Every consumer of a damage
  event dereferences the attacker — for relations, for blame, for the death log — and there
  is no sensible fallback identifier.
- The impact point within the bone is always the bone's origin. A scripted hit cannot place
  a wound precisely; it can only name the bone.
- The weapon field is always empty, so a scripted hit never counts toward a weapon's
  statistics and never triggers weapon-specific damage rules.

## Creature memory

**Contract** — the read-and-prune surface over what a creature remembers. The memory model
has four channels — what it has *seen*, what it has *heard*, what has *hit* it, and what it
considers *dangerous* — plus three running selections the brain maintains: the best enemy,
the best danger and the best item worth picking up.

Readers:

```text
best_enemy, best_danger, best_item      -> the current selections, or nothing
memory_time(object)                     -> when this creature last had evidence of it
memory_position(object)                 -> where it believes the object is
not_yet_visible_objects                 -> what it is partway through noticing
visibility_threshold                    -> how much evidence noticing requires
vision_enabled, is_there_items_to_pickup
```

Writers:

```text
enable_memory_object(object, enabled)   # exclude one object from being remembered at all
enable_vision(enabled)
set_sound_threshold(value), restore_sound_threshold
remove_danger(entry), remove_memory_sound_object(entry),
remove_memory_hit_object(entry), remove_memory_visible_object(entry)
```

**Invariants**

- `memory_position` is what the creature *believes*, not where the object is. The
  difference is the entire reason the memory system exists, and a rebuild that answers the
  true position removes the ability to flank, hide or break line of sight.
- The four pruners take a *memory entry*, not an object, because a creature can hold
  several entries about one object — seen here, heard there, hit from a third place — and
  forgetting one must not forget the others.
- The pruners fail **silently** on a non-stalker, while every reader logs. A rebuild
  should log both.
- `not_yet_visible_objects` and `visibility_threshold` **abort** rather than falling back,
  because one returns a reference into the creature's own table and the other is meaningless
  without one. This is the facade's documented third escalation level.
- Noticing is gradual: an object accumulates visibility until it crosses the threshold, and
  the partway list is what a script reads to make a creature react *while* it is being
  spotted. Lowering the threshold makes creatures sharp-eyed; the sound threshold is the
  same idea for hearing and has an explicit restore because scripts raise it temporarily
  for stealth sequences.

## Item handling: `set_item`

**Contract** — sets a stalker's standing goal for what to do with an item — the same
vocabulary as the [object action channel](script_object_action.h.md) — in four arities:
the order alone, the order and its subject, plus a burst size, plus a burst interval.

**Invariants** — where the three-argument form is given a burst size, it is passed as
**both** the minimum and the maximum, and the four-argument form likewise passes the
interval as both bounds. The underlying goal takes ranges; the script surface exposes only
exact values, collapsing the range. A rebuild may expose the ranges, and scripts that want
varied bursts currently cannot get them.

## `best_weapon`

**Contract** — the stalker's currently preferred weapon, or nothing. Checks not only that a
weapon was selected but that it is **still in this stalker's hands** — its container must
be this very entity.

**Notes**

The containment re-check exists because the selection can outlive the transfer: a weapon
dropped, taken or destroyed between the brain's choice and the script's question would
otherwise be handed back as "his best weapon" while lying on the floor. The same class of
staleness as the memory readers'.

## `explode`

**Contract** — detonates an explosive lying in the world. Refuses, with a script error, on
an object that has a container — a grenade in a pocket may not be detonated by script —
finds the surface normal beneath it, names the object itself as the initiator, and raises
the explosion event at its position.

**Invariants** — the containment check runs **before** the type check, so a contained
non-explosive reports the wrong problem. The self-initiation means a scripted explosion is
blamed on the object, not on whoever placed it.

## `play_cycle`

**Contract** — plays a named animation cycle on the object's skinned model, optionally
blending into it from the current pose rather than cutting. The blending form is the
default, because cutting between cycles is visible.

**Invariants** — two separate failures are reported distinctly: the object is not animated
at all, or it has no cycle by that name. Both log and do nothing.

## Actor and creature placement

**Contract** — `set_actor_position`, `set_npc_position` and `set_actor_direction`
teleport and re-aim. Placement writes the translation of the entity's transform and forces
it through, rather than assigning the transform, so the physics and collision
representations are updated with it.

```text
FUNCTION set_npc_position(position)
  IF not a creature THEN log script error; RETURN
  transform = this entity's transform with its translation replaced
  invalidate the creature's detail path             # it now starts somewhere else
  IF an animation is currently driving the transform THEN stop it
  force the transform
```

**Invariants** — the creature form does two things the actor form does not, and both are
load-bearing: the **detail path is invalidated**, because a path computed from the old
position would walk the creature back; and any **animation-driven motion is torn down**,
because an animation that owns the transform would immediately overwrite the teleport. A
rebuild that omits either produces a creature that snaps back or slides.

`set_actor_direction` turns the *camera*, not the body: the player's facing is the camera's,
and writing the body's transform would be overwritten on the next frame.

## Actor-only accessors

**Contract** — `disable_hit_marks` reads and writes whether damage flashes are drawn;
`movement_speed` reads the actor's current velocity and aborts on a non-actor;
`current_holder` returns the vehicle or mounted device the actor occupies, or nothing.

## Enemy-selection thresholds

**Contract** — two tunings on a stalker's enemy selection, each with a reader, a writer and
an explicit restore-to-default:

- `ignore_monster_threshold` — how badly a creature must threaten before the stalker turns
  from a human enemy to deal with it. The written value is clamped to the unit range.
- `max_ignore_monster_distance` — beyond this range the threshold does not apply at all.

**Notes**

The pair exists because stalkers fighting each other used to break off to shoot at distant
harmless dogs. The explicit restore on both — rather than expecting the script to remember
the old value — is what lets a scripted sequence raise the threshold and hand control back
cleanly.

## Dialogue entry points, zone contact, path lookahead, armour

**Contract** — `set_start_dialog`, `get_start_dialog` and `restore_default_start_dialog`
select which authored phrase graph a talking creature opens with; all three do nothing
silently on a non-talker. `active_zone_contact(id)` asks a scripted zone whether a given
entity is currently inside it. `location_on_path(distance, out_position)` reports the point
a given distance ahead along the creature's current detail path, answering an invalid
marker when asked without a destination or of a non-creature.
`reset_bone_protections(immunity_section, bone_section)` re-reads a stalker's per-bone
armour from two named configuration sections, which is how an outfit change takes effect.
`set_visual_name` and `get_visual_name` swap the model a live object is drawn with.

**Notes**

`get_start_dialog` returns nothing at all — it calls the underlying reader and discards its
answer. Either the signature is wrong or the call is vestigial; a rebuild should return the
identifier.
