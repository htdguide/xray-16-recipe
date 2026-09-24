# src/xrServerEntities/script_value_container_impl.h

> The shadow-cell container's three operations: duplicate-rejecting insert, batch write-back, destroy-all.

**Needs** — [`script_value_container.h`](script_value_container.h.md) · [`script_value.h`](script_value.h.md)
**Used by** — [`script_properties_list_helper.cpp`](script_properties_list_helper.cpp.md) · [`script_value_container.h`](script_value_container.h.md) · [`xrServer_Object_Base.cpp`](xrServer_Object_Base.cpp.md) · [`xrServer_Objects_Alife_Smartcovers.cpp`](xrServer_Objects_Alife_Smartcovers.cpp.md)
**Tier floor** — T2.

## Purpose

Holds the bodies the container's header declares. It is a separate file from the header for
one build reason worth stating: the header is included by every record, but the bodies need
the cell's full definition, so only the few units that actually construct cells pay for it.
In a rebuild that distinction vanishes and this merges into the container.

## `add`

**Contract** — takes ownership of a cell. If a cell with the same name is already present,
the new cell is **dropped on the floor** — not replaced, not reported, and the caller's
pointer leaks.

```text
FUNCTION add(container, cell)
  IF ANY c IN container.cells WHERE c.name == cell.name
    RETURN                    # the existing cell wins; the new one is abandoned
  APPEND cell TO container.cells
```

**Notes** — the leak is real but bounded: cells are created during record construction from
a fixed field list, so a duplicate means a script declared the same field name twice, which
is a script bug that will also produce nonsense at serialization time. A rebuild should
*fail loudly* here instead; nothing depends on the silence.

The scan is linear over the whole list per insertion. Field counts are single digits, so
this is the right shape — a map would cost more than it saves.

## `assign`

**Contract** — calls every cell's write-back in insertion order. Never fails; a cell whose
script table has gone away is the script layer's problem, not this one's.

## `clear`

**Contract** — destroys every cell and empties the list. Called from destruction, and
callable directly when a record is being rebuilt from a different script class.
