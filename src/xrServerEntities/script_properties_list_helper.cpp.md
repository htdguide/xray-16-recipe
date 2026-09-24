# src/xrServerEntities/script_properties_list_helper.cpp

> Makes a field of a script table editable as if it were a field of a record, by giving the editor a real memory location and keeping it synchronized with the table.

**Needs** — [`script_properties_list_helper.h`](script_properties_list_helper.h.md) · [`script_value_wrapper.h`](script_value_wrapper.h.md) · [`script_value_container_impl.h`](script_value_container_impl.h.md) · [`xrServer_Object_Base.h`](xrServer_Object_Base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

The editor's property model binds to a **pointer to a field**
([`PropertiesListTypes.h`](PropertiesListTypes.h.md)). A script class's state is a table
entry, which has no address. This file resolves that mismatch, and the resolution — two
different strategies depending on the field's type — is the only decision in an otherwise
mechanical file of two hundred forwarding calls.

## `bind_field`

**Contract** — given a script object and a field name, produce a pointer the editor can
write through, for the lifetime of the record.

```text
FUNCTION bind_field(object, field_name) -> pointer to value
  IF the field's type is an engine class that scripts hold by reference
     (a vector, a flag word, a colour — but not an interned string)
    RETURN the address of the value already living inside the script table
  ELSE
    w = new shadow cell holding a copy, remembering (object, field_name)
    owning_record(object).adopt(w)      # the record keeps it alive and writes it back
    RETURN address of w
```

**Invariants**

- The owning record must be reachable from the script object — the binding asserts it.
  Every script entity class descends from the record base, so the assertion holds by
  construction; it fires when a script passes something that is not an entity.
- A shadow cell is owned by the record, not by the property list, so it outlives the editor
  panel. When the record serializes, the cells are written back into the script table. That
  is what the value-container machinery in
  [`script_value_container.h`](script_value_container.h.md) exists for.

**Notes** — the split is between types a script holds **by reference** and types it holds
**by value**. A vector in a Lua table is a userdata the engine owns, so its address is
stable and can be handed straight to the editor. A number or a string is copied in and out
of the table on every access, so there is no address to hand over and a shadow cell is the
only option. The interned-string type is explicitly excluded from the by-reference case
despite being a class, because assigning to one has to go through the intern table.

There is a second binding form that takes the object *and* a separate table, used for the
boolean property: the value lives in a different table from the record. It exists for one
call site.

## the `create…` family

**Contract** — each call binds the named field and forwards to the corresponding native
property factory in [`PropertiesListHelper.h`](PropertiesListHelper.h.md), with the same
minimum/maximum/step/decimals arguments. There is no logic in any of them beyond the
binding; the overload count is the arity ladder that the script binding layer needs in order
to expose optional arguments.

## the three edit hooks

**Contract** — forward the vector, float and name edit hooks to the native ones, converting
the interned-string type at the boundary in both directions. The name hook's conversion is
the reason it cannot simply be exposed directly.
