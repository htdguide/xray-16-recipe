# src/xrGame/relation_registry_defs.h

> The stored shape of one character's opinions: a goodwill number per other character and per faction.

**Needs** — [`Common/object_interfaces.h`](../Common/object_interfaces.h.md)
**Used by** — [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · [`relation_registry.h`](relation_registry.h.md)
**Tier floor** — T2: two sparse maps, serialized into the save

## Purpose

Declares what the relation registry actually persists. The registry itself
([`relation_registry.cpp`](relation_registry.cpp.md)) computes attitudes from several
sources; only the two maps declared here are *stored*, and everything else is derived on
each query. Keeping that distinction visible is the reason this is a separate file.

## State

```text
RECORD Relation
  goodwill : int        # signed; the stored opinion. Defaults to the neutral value,
                        #   never to "unset" — an absent entry and a neutral entry
                        #   are indistinguishable by design

RECORD RelationData                        # one per character that has ever had an opinion
  personal    : map<entity id, Relation>   # what THIS character thinks of that character
  communities : map<faction id, Relation>  # what that FACTION thinks of THIS character
```

**Invariants**

- The two maps face in **opposite directions**, and this is the single most confusing thing
  in the relation system. `personal` is indexed by the *target*: it is the owner's opinion
  of others. `communities` is indexed by the *source*: it is how each faction feels about
  the owner. A character therefore carries its own reputation with every faction, rather
  than each faction carrying a roster. The reason is storage: a faction has no entity
  record to hang a map on, and a character does.
- A missing entry reads as neutral. There is no "no opinion" state at this level, so nothing
  can distinguish a character who was never met from one deliberately set to neutral.
- Both maps are sparse: an entry appears only when something writes one.

## `RelationData` — clear, load, save

**Contract** — implements the shared serialization interface. `clear` empties both maps;
`save` writes the personal map then the community map using the generic map serializer;
`load` reads them back in the same order. Positional, with no tags — the field order is the
format, and swapping the two maps invalidates every existing save.

**Notes** — these records are stored in the alife registry keyed by entity identifier, which
is what makes opinions survive an entity going offline. They are not attached to the client
object and do not need the character to be loaded.
