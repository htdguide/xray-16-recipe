# src/xrCommon/xr_map.h

> The engine's ordered key-to-value table, allocating through the engine allocator, with the key ordering left open.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md) · [`predicates.h`](predicates.h.md)
**Used by** — [`graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`xr_shared.h`](../xrCore/xr_shared.h.md)
**Tier floor** — T3: an allocator injection point, plus one decision about ordering.

## Purpose

Exists to inject an allocator — but unlike the sequence aliases it also fixes a *default
ordering*, and that default is the load-bearing part. Two forms are named: one rejecting
duplicate keys, one admitting them.

## `xr_map` / `xr_multimap`

**Contract** — a table mapping keys to values, iterated in key order, parameterised by the
comparison that defines that order. The default comparison is the key type's own natural
"less than". The duplicate-admitting form keeps equal keys adjacent in insertion order.

**Invariants**

```text
# The comparison is a strict weak ordering and iteration order is that ordering.
#   Several load paths iterate a table and write the result to a file or hand it
#   to the renderer in iteration order, so the ordering is observable in output,
#   not just an implementation detail. Swapping in a hash table changes results.

# When the key is a raw text pointer, the DEFAULT ordering compares POINTERS, not
# characters - and is therefore run-to-run unstable and almost never what was
# meant. Every such table in the engine must pass an explicit text comparison;
# see predicates.h. A rebuild whose text type compares by content removes this
# trap along with the file.
```

## Declaration shorthands

**Contract** — four macro forms declaring a named table type with its iterator type:
derived-iterator-name, given-iterator-name, given-iterator-name-with-explicit-ordering, and
the duplicate-admitting variant. Only the third carries information — it is the marker that
a table chose a non-default ordering. A rebuild deletes all four and writes the ordering at
the declaration.
