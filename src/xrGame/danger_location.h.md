# src/xrGame/danger_location.h

> The interface a "dangerous place" must satisfy: a position, a lifetime, a radius, and the set of squad members it warns.

**Needs** — [`memory_space.h`](memory_space.h.md) · [`danger_location_inline.h`](danger_location_inline.h.md)
**Used by** — [`agent_location_manager.cpp`](agent_location_manager.cpp.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`agent_location_manager_inline.h`](agent_location_manager_inline.h.md) · [`danger_cover_location.h`](danger_cover_location.h.md) · [`danger_cover_location_inline.h`](danger_cover_location_inline.h.md) · [`danger_location.cpp`](danger_location.cpp.md) · [`danger_location_inline.h`](danger_location_inline.h.md) · [`danger_object_location.h`](danger_object_location.h.md)
**Tier floor** — T3: an abstract record with a lifetime rule

## Purpose

A *danger location* is shared, squad-level knowledge: "this spot is bad, for these members,
until this time". It is distinct from a [danger object](danger_object.h.md), which is one
creature's private perception of a threat. Locations are what the group's path planning
and cover selection avoid; objects are what one brain reacts to.

This is an abstract base and therefore substantive: what it demands of an implementor is
the contract a rebuild must satisfy. Exactly one implementation ships in this directory,
[`CDangerObjectLocation`](danger_object_location.h.md), which pins the location to a moving
game object; others exist elsewhere for fixed points such as grenade landing spots.

## State

```text
RECORD DangerLocation                      # abstract; reference-counted and shared
  level_time : int      # when the location was recorded
  interval   : int      # how long it stays valid after that, in the same units
  radius     : real     # how far from the position the warning extends
  mask       : squad_mask   # bit per squad member; who is warned by this location
```

**Invariant** — the record is shared by reference between the squad's members and must not
be mutated by a reader. Its lifetime is owned by a reference count rather than by any one
member, because the member who recorded it may die before it expires.

**Invariant** — `mask` is a bitset the width of a squad, not a list of identities. Squad
membership is therefore an *index*, and a location outlives a change of membership with the
wrong bits set. The squad brain is responsible for re-issuing locations when the roster
changes.

## `position`

**Contract** — abstract; every implementor must answer where the location is. It is a
function rather than a field because the location may be attached to something that moves.

## `useful`

**Contract** — whether the location should still be considered. The default rule is pure
expiry: still useful while the global clock has not passed `level_time + interval`. An
implementor may override to something that never expires.

```text
FUNCTION useful() -> bool
  RETURN NOT (now > level_time + interval)
```

**Notes** — the comparison is against the engine's global millisecond clock while the field
is named for the level clock. Every constructing caller in this directory passes the global
clock, so the two agree in practice; the naming is the residue of an earlier design. A
rebuild should name one clock and use it.

## Match against a position

**Contract** — a location equals a position when the two are *similar* — within the math
layer's epsilon, not bit-exact. Danger locations are looked up by "is there already one
here", and an exact comparison would let two near-identical warnings coexist.

## Match against a game object

**Contract** — the base answers *no* to every object. A location is a place, and only an
implementor that knows it is bound to an entity can claim to be about that entity. This is
the hook the object-destroyed sweep uses to drop locations that named a dead entity.

## `mask`

**Contract** — the warned-member bitset, read-only.
