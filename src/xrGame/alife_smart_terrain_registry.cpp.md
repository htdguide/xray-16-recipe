# src/xrGame/alife_smart_terrain_registry.cpp

> Membership of the smart-terrain index: which server objects are places that hand out jobs.

**Needs** — [`alife_smart_terrain_registry.h`](alife_smart_terrain_registry.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a map insertion behind a kind test

## Purpose

The registration routines call this for *every* entity, and it keeps the ones that are
smart terrains. That is the whole file: a filtered index, so that a creature looking for
work does not scan the object registry.

## `add`

**Contract** — offered every registered entity; keeps only smart zones. Fails hard on a
duplicate identifier.

```text
FUNCTION add(object)
  IF object is not a smart zone -> RETURN
  REQUIRE object.id not already present
  objects.insert(object.id, object)
```

## `remove`

**Contract** — the mirror; ignores anything that is not a smart zone, and fails hard if a
smart zone is not present.

**Invariants** — note the asymmetry with the other alife registries: there is **no
tolerate-missing escape** on either side here. A smart terrain's membership depends only
on its kind, which never changes, so absence on removal is always a fault — unlike the
schedule registry, whose membership predicate can flip. A rebuild should keep the check
strict for exactly that reason.
