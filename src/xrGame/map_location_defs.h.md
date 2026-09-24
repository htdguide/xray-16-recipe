# src/xrGame/map_location_defs.h

> The saved form of a map marker: the (spot type, object) key that identifies it, and the per-level registry that persists them.

**Needs** — [`alife_abstract_registry.h`](alife_abstract_registry.h.md) · [`map_location.h`](map_location.h.md)
**Used by** — [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`map_manager.cpp`](map_manager.cpp.md) · [`map_manager.h`](map_manager.h.md) · [`UITaskWnd.cpp`](ui/UITaskWnd.cpp.md)
**Tier floor** — T3: a record and its serialization

## Purpose

A map marker is identified by *what kind* of marker it is and *which entity* it follows —
not by a handle, because the same entity can carry several markers of different kinds, and
because the pair is what a script and a save file can both name. This file holds that key,
the ordering that keeps stale markers out of the way, and the registry that writes the
serializable ones into the save.

## State

```text
RECORD LocationKey
  spot_type : text            # names an entry in the map-spot description data
  object_id : int (16-bit)    # the entity this marker follows
  location  : optional<MapLocation>   # the live marker; none once it is torn down
  actual    : bool = true     # false once the marker's subject is gone

RECORD MapLocationRegistry : per-level map of level identifier -> list<LocationKey>
```

**Invariant** — keys are ordered so that **every stale key sorts after every live one**, and
live keys are ordered among themselves by the identity of the marker they hold. The manager's
per-frame walk relies on this: it can stop at the first stale key rather than filtering, and
removal is a truncation from the tail.

Ordering live keys by marker identity rather than by spot type or entity is what makes the
order *stable* while nothing is created or destroyed — it is not a meaningful order, just a
consistent one, and nothing may depend on which live key comes first.

**Invariant** — the registry is keyed by **level**, so a marker is restored only on the level
it belongs to. The alife simulation moves entities between levels; a marker whose entity has
left is restored under the new level's list when the entity is written there.

## `save` · `load`

**Contract** — a key writes its spot type and its entity identifier, then asks its marker to
write itself. Loading rebuilds the marker from the spot type and the entity and then lets it
read its own state back. Only keys whose marker is marked serializable are ever written — the
registry filters them out on the way out, not on the way in.

**Invariant** — the order is spot type, entity identifier, marker payload, and it is frozen:
save files from the shipped games are read with it.

## `destroy`

**Contract** — releases the marker the key holds. The key is a value in a vector, so its own
lifetime is the vector's; the marker it points at is owned and must be released explicitly.
This is the incidental half of the file — a rebuild whose container owns its elements needs
no such call.

## `MapLocationRegistry::save`

**Contract** — writes the registry, having first discarded every key whose marker is not
serializable. Non-serializable markers are the runtime annotations scripts create and the
relation markers the game generates every frame; writing them would grow the save without
bound and restore markers for entities that no longer justify them.
