# src/xrGame/agent_corpse_manager_inline.h

> Registration and access for the squad's pending-death list.

**Needs** — [`agent_corpse_manager.h`](agent_corpse_manager.h.md) · [`member_corpse.h`](member_corpse.h.md)
**Used by** — [`agent_corpse_manager.h`](agent_corpse_manager.h.md)
**Tier floor** — T3: list operations

## Purpose

The small half of the corpse manager: construction, the squad accessor, and the two list
operations that are not part of the matching algorithm in
[`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md).

## State

Adds nothing beyond what the header declares.

## Construction

**Contract** — binds the squad brain this manager belongs to; requires it to be present.
The binding is permanent.

## `register_corpse`

**Contract** — appends a pending entry for a fallen squad member, with no reactor and
stamped with the current global clock. **Requires that the same body is not already
pending** — a duplicate registration is a contract violation, not a tolerated repeat,
because it would let two members be assigned to the same death.

**Notes** — the duplicate check is a linear scan compiled out of release builds, so in a
release build a double registration silently produces a duplicated reaction. The list
holds at most a squad's worth of entries, so a rebuild can afford to keep the check
always.

## `corpses` / `clear`

**Contract** — the pending list, writable, and its reset. The writable exposure exists
because the squad brain's other managers consult the list; a rebuild should narrow it to a
read-only view plus explicit mutators.

## `object`

**Contract** — the squad brain; requires the binding.
