# src/xrGame/item_manager_inline.h

> Dead code: an inline constructor that the implementation file defines again and overrides.

**Needs** — [`item_manager.h`](item_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: unreachable

## Purpose

Holds an inline version of the item manager's constructor that stores the owning creature.
The real constructor in [`item_manager.cpp`](item_manager.cpp.md) does the same thing *and*
resolves the stalker form of the owner; the declaration header never includes this file, so
this definition is not reachable. It is left over from a split that was undone.

A rebuild does not create this file. It is recorded here only so a reader comparing the
mirror against the source does not go looking for a missing page.

## State

`Stateless.`
