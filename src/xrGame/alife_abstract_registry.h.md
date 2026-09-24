# src/xrGame/alife_abstract_registry.h

> The shape every alife registry shares: a keyed table of records that saves and loads with the game, and that asserts loudly when a key is added twice or looked up and missing.

**Needs** — [`alife_abstract_registry_inline.h`](alife_abstract_registry_inline.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`LevelFogOfWar.h`](LevelFogOfWar.h.md) · [`actor_statistic_defs.h`](actor_statistic_defs.h.md) · [`alife_abstract_registry_inline.h`](alife_abstract_registry_inline.h.md) · [`alife_registry_container.cpp`](alife_registry_container.cpp.md) · [`alife_registry_container.h`](alife_registry_container.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`alife_registry_wrapper.h`](alife_registry_wrapper.h.md) · [`map_location_defs.h`](map_location_defs.h.md)
**Tier floor** — T3: a keyed table with a serialization contract.

## Purpose

The off-screen simulation keeps several indexes over its world — which entities are on
which cross-level graph vertex, which are on which level, which belong to which group.
They differ only in what they are keyed by and what they hold. This file is the shape they
share, so that each concrete registry is a key type, a record type and whatever it adds on
top.

The one substantive decision here is the *error policy*, and it is the same for every
registry: adding a duplicate key or reading a missing one is a bug by default, reportable
by the caller as tolerable case by case.

## State

```text
RECORD AlifeRegistry<Key, Record>
  objects : map<Key, Record>      # ordered by key: the save format depends on it

# Invariant: the map's iteration order is the key's order, and the save writes it in
#   that order. A rebuild using an unordered table must sort before writing, or saves
#   stop being byte-comparable across runs.
# Invariant: the registry owns its records. Destroying it destroys them, which is why
#   every concrete registry is destroyed exactly once at simulation teardown.
```

## `add`

**Contract** — Inserts a record under a key. A key already present is a hard error by
default and the insert is skipped; the caller may pass a flag that downgrades the error to
a silent skip, for the paths where a duplicate is a legitimate race with the level's own
spawn.

## `remove`

**Contract** — Erases the record under a key. A missing key is a hard error by default,
downgradable the same way. Removal destroys the record.

## `object`

**Contract** — Returns a reference to the record under a key, or *absent* when the key is
missing; missing is a hard error by default and downgradable. The reference is into the
table and is invalidated by any later insertion or removal.

## `save` / `load`

**Contract** — Serializes the whole table through the engine's generic record
serialization, which writes the entry count followed by each key and record in iteration
order. Loading replaces whatever the table held.

**Notes** — The registries are part of the save game (§5, frozen only against itself), so
the record layout is whatever the record type declares. The registry itself contributes
only the count and the ordering, both of which a rebuild is free to redesign as long as it
does so consistently across every registry, since they share this one implementation.
