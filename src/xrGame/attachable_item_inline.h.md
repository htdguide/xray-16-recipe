# src/xrGame/attachable_item_inline.h

> The trivial accessors and the initial state of the attachable-item mix-in.

**Needs** — [`attachable_item.h`](attachable_item.h.md)
**Used by** — [`attachable_item.h`](attachable_item.h.md)
**Tier floor** — T3: field reads

## Purpose

Splits the one-line accessors of
[`attachable_item.h`](attachable_item.h.md) out of the declaration so they can be inlined at
every call site. Purely a compilation arrangement: a rebuild puts these where the fields
are and has no second file.

## State

Declares no state of its own. It fixes the *initial* state, which is the one substantive
thing here: no linked inventory item, an identity offset, an empty bone name, and
**enabled true**.

**Notes** — starting enabled and being switched off by the load path (see
[`attachable_item.cpp`](attachable_item.cpp.md)) means an item whose section declares no
attach position stays "enabled" forever. That is harmless — every path that acts on the flag
first checks for a carrier that can hold attachments — but it means the flag alone does not
answer "is this hanging", and a rebuild reading the flag in isolation will be wrong.

## accessors

**Contract** — `bone_name`, `offset`, `bone_id`, `set_bone_id` and `enabled` read and write
the corresponding fields; `item` returns the linked inventory-item half. All but `enabled`
assert in debug builds that the attach position has been loaded, which is how a rebuild
learns the ordering requirement: nothing may read the bone or the offset before the item's
section has been read.
