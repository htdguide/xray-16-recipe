# src/xrServerEntities/script_value_container.h

> The set of shadow cells one record owns, written back to the script table as a batch.

**Needs** — [`script_value.h`](script_value.h.md) · [`script_value_container_impl.h`](script_value_container_impl.h.md)
**Used by** — [`script_value.h`](script_value.h.md) · [`script_value_container_impl.h`](script_value_container_impl.h.md) · [`xrServer_Object_Base.h`](xrServer_Object_Base.h.md) · [`xrServer_Objects_Alife_Smartcovers.h`](xrServer_Objects_Alife_Smartcovers.h.md)
**Tier floor** — T2: an owning collection with a defined destruction point.

## Purpose

Declares the surface every script-declared record mixes in: a collection of shadow cells,
plus "write all of them back". A record that is native has no container of its own worth
speaking of; a record declared in script has one cell per field it wants serialized or
edited, and the container is what turns "this record's script state" into a single
operation the serializers can call.

The algorithms are in [`script_value_container_impl.h`](script_value_container_impl.h.md).

## State

```text
RECORD ScriptValueContainer
  cells : list<ScriptValue>   # owned; insertion order is write-back order
```

**Invariants** — no two cells share a name. Cell lifetime is the container's lifetime: the
container destroys every cell it holds, which is why cells are handed over rather than
borrowed.

## the exported surface

`add` (take ownership of a cell, ignoring a duplicate name), `assign` (write every cell
back, in insertion order) and `clear` (destroy every cell). Destruction clears.

## Notes

**Write-back order is insertion order and is observable**, because two cells may name the
same table through different paths. Nothing in the engine relies on it today; a rebuild
should keep it anyway, since the alternative — a name-keyed map — silently reorders and
would be a behaviour change no test catches.
