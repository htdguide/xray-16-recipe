# src/xrGame/alife_schedule_registry_inline.h

> One pass of the offline round-robin, and the budget it is bounded by.

**Needs** — [`alife_schedule_registry.h`](alife_schedule_registry.h.md)
**Used by** — [`alife_schedule_registry.h`](alife_schedule_registry.h.md)
**Tier floor** — T2: a bounded walk over a map

## Purpose

Holds the pass itself, which the header's declaration only names. Membership rules are in
[`alife_schedule_registry.cpp`](alife_schedule_registry.cpp.md).

## `update` — one pass

**Contract** — advances up to `objects_per_update` scheduled objects, starting where the
previous pass stopped. Does nothing when the set is empty. Each advanced object's own
update may add to or remove from this set, including removing itself, and the pass must
survive that.

```text
FUNCTION update()
  IF objects.is_empty() -> RETURN
  cycle_count = cycle_count + 1
  advanced = 0
  WHILE advanced < objects_per_update
    entry = objects.at(cursor)
    IF entry.schedule_counter == cycle_count
      BREAK                       # wrapped: everything reachable has already run this pass
    entry.schedule_counter = cycle_count
    advanced = advanced + 1
    advance cursor                # BEFORE the update, so the update may remove this entry
    entry.update()
```

**Invariants** —

- The cursor is advanced **before** the object's update runs. That single ordering is what
  makes the pass safe against an object removing itself — a creature dying, a squad
  dissolving, an anomaly being consumed — which is a routine occurrence, not an edge case.
- The counter is stamped before the update, for the same reason: an object that removes
  and re-adds itself must not be advanced a second time in the same pass.
- The set is keyed and ordered by entity identifier, so the rotation order is by
  identifier. Nothing depends on it being that order, only on it being stable.

## `objects_per_update`

**Contract** — reads and writes the per-pass budget. Set to one at construction; raised
by the alife update manager when the simulation is behind and lowered when it catches up.
Zero means the simulation stops advancing, which is a legitimate state during a load.

## `object`

**Contract** — resolves an identifier to the scheduled object. Absence is a fault unless
the caller tolerates it, as everywhere else in the alife registries.
