# src/xrServerEntities/script_value.h

> One shadow cell: a copy of a named field of a script table, owned by a record so the field has a stable address.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [`script_value_inline.h`](script_value_inline.h.md) · [`script_value_container.h`](script_value_container.h.md)
**Used by** — [`script_value_container.h`](script_value_container.h.md) · [`script_value_container_impl.h`](script_value_container_impl.h.md) · [`script_value_inline.h`](script_value_inline.h.md) · [`script_value_wrapper.h`](script_value_wrapper.h.md)
**Tier floor** — T2: the whole point is an addressable, typed cell that outlives the script value it mirrors.

## Purpose

A record declared in script keeps its state in a script table. Two parts of the engine need
that state at a *fixed address* instead: the serializers, which write fields into a byte
stream, and the editor's property model, which hands out typed handles onto a field so the
user can edit it in place. A script table field has neither a fixed address nor a static
type.

This declares the abstraction that bridges the gap — a cell that holds a copy of one named
field, remembers which table and which name it came from, and knows how to push its copy
back. The base is deliberately untyped: what the cell *holds* is the subclass's business
(see [`script_value_wrapper.h`](script_value_wrapper.h.md)); what every cell must do is
write back.

## State

```text
RECORD ScriptValue
  table : script_object    # the script table the field lives in
  name  : text             # the key within it; also the cell's identity in its container
```

**Invariants** — the name is unique within one container, which is what lets the container
reject duplicates by name rather than by address. The table reference keeps the script
table alive for as long as the cell exists.

## `ScriptValue`

**Contract** — constructed from a script table and a field name. Construction is where the
initial read happens in every concrete subclass: the cell is *seeded* from the table, so a
cell always starts out agreeing with the script side.

## `assign`

**Contract** — writes the cell's current copy back into the table under its name. Every
subclass must supply it; there is no meaningful default. It is called for the whole
container at once, immediately before serialization and after an edit, and never
per-field — that batching is the reason the container exists.

## `name`

**Contract** — the field name, interned. Used only for duplicate detection.

## Notes

**The copy is one-way at rest.** Between construction and `assign` the cell and the table
can disagree: the engine edits the cell, the script edits the table, and whoever writes back
last wins. Nothing detects the conflict. A rebuild that makes the cell a live view of the
table field instead would be *more* correct, but the native side needs a raw address for the
editor's property handles and for the serializers, which is exactly what a live view cannot
give.
