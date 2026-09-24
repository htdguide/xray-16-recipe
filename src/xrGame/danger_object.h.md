# src/xrGame/danger_object.h

> One recorded threat perception: who, where, when, what kind, and through which sense it arrived.

**Needs** — [`entity_alive.h`](entity_alive.h.md) · [`danger_object_inline.h`](danger_object_inline.h.md)
**Used by** — [`danger_manager.h`](danger_manager.h.md) · [`danger_object.cpp`](danger_object.cpp.md) · [`danger_object_inline.h`](danger_object_inline.h.md) · [`memory_space_script.cpp`](memory_space_script.cpp.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`stalker_danger_property_evaluators.cpp`](stalker_danger_property_evaluators.cpp.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md)
**Tier floor** — T3: a value record and an equality rule

## Purpose

The atom of the danger system. A creature's perception layer produces these; the
[danger manager](danger_manager.cpp.md) collects, de-duplicates, ages and ranks them, and
the brain's motivation weighting reads the winner. This is a header with a near-empty
implementation file, so the substance is here.

The type is a plain value, copied into and out of a list. That is deliberate: a danger is
a *snapshot* of a perception, not a live view of the world. The entity it names may die or
be destroyed after the record is made, which is why the manager has an explicit
link-clearing path rather than relying on the record staying valid.

## State

```text
RECORD DangerObject
  object           : optional<EntityAlive>   # the living entity the danger is about;
                                             #   absent for a sound with no known owner
  dependent_object : optional<GameObject>    # the thing that produced it — the grenade,
                                             #   the projectile. Cleared when it dies.
  position         : vector                  # where the perception placed it, frozen at
                                             #   record time; not tracked afterwards
  time             : int                     # the level clock at perception
  type             : DangerType
  perceive_type    : PerceiveType
```

```text
ENUM DangerType
  bullet_ricochet       # a bullet or blade struck near me
  attack_sound          # someone is shooting
  entity_attacked       # someone took a hit
  entity_death          # someone died
  fresh_entity_corpse   # I can see a body that was killed
  attacked              # I registered a hit landing on another entity
  grenade               # a grenade is about to go off nearby
  enemy_sound           # an enemy made any sound at all

ENUM PerceiveType
  visual · sound · hit
```

**Invariant** — the pair `(entity identity, type, perceive_type)` is the record's identity.
Two records comparing equal are the *same danger observed again*, and the manager replaces
rather than appends. Position and time are explicitly *not* part of identity, so a repeated
observation refreshes both — this is what makes a continuously-firing enemy one danger with
a rolling timestamp instead of a list that grows every frame.

**Invariant** — an absent entity matches only an absent entity. A nameless sound danger
never merges with a named one even when the type and sense agree.

## `DangerObject` construction

**Contract** — takes entity, position, time, type, perceive type and an optional dependent
object; stores all six verbatim. No validation, no clock reading: the caller supplies the
timestamp, because the perception event carries the level time at which it was *perceived*,
not the time the record is built.

## Equality

**Contract** — total, no failure mode. Compares on identity as described above. Note that
the comparison reaches for the entity's identifier rather than its address, so a record
still matches after the entity has been reloaded from a save under a new address.

## Accessors · `clear_dependent_object`

**Contract** — read-only views of every field, plus one mutator: dropping the dependent
object. That mutator exists for exactly one caller — the manager's link-clearing path, run
when a game object is destroyed — and it must be able to run on a record that is otherwise
still useful. A danger whose grenade has already exploded is still a danger.
