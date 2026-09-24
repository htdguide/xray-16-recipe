# src/xrGame/alife_schedule_registry.cpp

> Membership rules for the offline update rotation: which server objects get an alife tick at all.

**Needs** — [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md)
**Used by** — reached through its declarations in [`alife_schedule_registry.h`](alife_schedule_registry.h.md); callers name that, not this file.
**Tier floor** — T2: a membership test plus a map insertion

## Purpose

The scheduler for offline entities is a round-robin over a set; this file decides what is
in that set. Two independent conditions must hold, and both are re-tested on the way out
as well as on the way in.

Substance of the rotation itself is in
[`alife_schedule_registry_inline.h`](alife_schedule_registry_inline.h.md) and in the
generic round-robin it builds on.

## `add`

**Contract** — offers a server object to the rotation. Silently ignores an object that
does not qualify. Fails hard if the object is already present.

```text
FUNCTION add(object)
  IF object is not schedulable   -> RETURN     # not every server object has an offline tick
  IF NOT object.need_update()    -> RETURN     # it is schedulable but currently has nothing to do
  rotation.insert(object.id, object)
```

**Invariants** — the two rejections mean different things and a rebuild should keep them
apart. *Not schedulable* is a property of the entity's kind, fixed for its lifetime: an
item on the ground does not think. *Does not need updating* is a property of its current
state and may change: a creature that is part of a squad defers to the squad, an
anomaly that is dormant has nothing to advance.

## `remove`

**Contract** — withdraws an object from the rotation. Silently ignores a non-schedulable
object. Absence is a fault unless the caller tolerates it, **or** unless the object does
not currently need updating.

```text
FUNCTION remove(object, tolerate_missing)
  IF object is not schedulable -> RETURN
  rotation.remove(object.id,
                  tolerate_missing OR NOT object.need_update())
```

**Invariants** — the second half of the tolerance is the load-bearing line, and it is the
reason this file is not simply two delegations. An object whose `need_update` is false was
never inserted, so its removal must not be an error — but the caller does not know that,
because `need_update` may have flipped since. Re-evaluating the same predicate on the way
out is what keeps the pair symmetric without the caller tracking membership.

That symmetry has a sharp edge a rebuild inherits: if `need_update` flips from false to
true while an object is out of the rotation, nothing re-offers it; and if it flips from
true to false while the object is in, the removal is *tolerated* but the object is still
removed, so the states do stay consistent. The failure case is one-directional — an
object can silently miss being scheduled, never be scheduled twice — which is the safe
direction, and is presumably why it was left this way.

## Notes

An object's identifier is the key, so the rotation holds at most one entry per entity, and
the duplicate check in the underlying container is the same runtime invariant the object
registry enforces. Together they are the "no entity is registered twice" guarantee that
conformance §6 asks for, stated once per registry.
