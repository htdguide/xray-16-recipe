# src/Include/editor/property_holder_base.hpp

> The property-grid contract: how the engine describes one editable object to the editor — a list of typed, categorized, live-bound fields.

**Needs** — [`ide.hpp`](ide.hpp.md) · [`engine.hpp`](engine.hpp.md)
**Used by** — [`engine.hpp`](engine.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`property_boolean.cpp`](../../editors/xrWeatherEditor/property_boolean.cpp.md) · [`property_collection_base.cpp`](../../editors/xrWeatherEditor/property_collection_base.cpp.md) · [`property_collection_editor.cpp`](../../editors/xrWeatherEditor/property_collection_editor.cpp.md) · [`property_collection_enumerator.cpp`](../../editors/xrWeatherEditor/property_collection_enumerator.cpp.md) · [`property_holder.cpp`](../../editors/xrWeatherEditor/property_holder.cpp.md) · [`property_holder.hpp`](../../editors/xrWeatherEditor/property_holder.hpp.md) · [`property_holder_include.hpp`](../../editors/xrWeatherEditor/property_holder_include.hpp.md) · [`property_integer_values_value_getter.hpp`](../../editors/xrWeatherEditor/property_integer_values_value_getter.hpp.md) · [`property_integer_values_value_reference_getter.hpp`](../../editors/xrWeatherEditor/property_integer_values_value_reference_getter.hpp.md) · [`property_string_values_value_getter.hpp`](../../editors/xrWeatherEditor/property_string_values_value_getter.hpp.md) · [`property_string_values_value_shared_str_getter.hpp`](../../editors/xrWeatherEditor/property_string_values_value_shared_str_getter.hpp.md) · [`property_vec3f_base.hpp`](../../editors/xrWeatherEditor/property_vec3f_base.hpp.md) · _and 33 more_
**Tier floor** — T1: two of its records cross a module boundary between two different implementation languages with a declared four-byte packing, and every binding is a callable whose representation both sides must agree on.

## Purpose

Almost everything the weather editor edits is a flat list of named values — a keyframe's fog density, its ambient colour, its sky texture, its sun direction. The editor has one widget for all of it: a property grid with categories, descriptions, per-type cell editors and an undo-to-default. This file is how the engine tells that grid what to show.

The central decision: **a property is not a value the grid holds, it is a binding to the value where it already lives.** The engine passes a getter and a setter (or a direct reference to the field); the grid reads on paint and writes on edit. Nothing is copied, nothing is synchronized, and there is no apply step.

## State

```text
RECORD PropertyHolder                # one editable object
  display_name : text
  owner        : optional<PropertyOwner>        # the object that contains this one
  collection   : optional<PropertyCollection>   # the list this is an element of
  properties   : list<PropertyValue>

RECORD Color                         # layout is declared, not inferred
  r, g, b : real

RECORD Vec3                          # likewise
  x, y, z : real
```

**Invariants** — a holder's properties are added once, after creation, before the grid binds to it; `clear` removes them all and the holder can then be refilled. Identifier strings are unique within a holder. The holder does not own the values it binds to — it must be destroyed before they are.

**Notes** — `Color` and `Vec3` are declared here, tiny and redundant with the engine's own vector types, with an explicit four-byte packing. That is not duplication for its own sake: these two records are *passed by value across a boundary between two separately compiled modules written in different languages*, and neither side may assume the other's layout rules. The engine's own vector types carry alignment chosen for wide-float math and cannot cross safely. A rebuild that keeps the split boundary needs the same two declarations; one that merges the editor into the engine deletes them.

## The property vocabulary

The interface is one heavily overloaded add-a-property call. The overload set is the real content of the file, so it is given here as the grid of decisions it encodes rather than as sixty signatures.

Every property, whatever its type, carries:

```text
identifier   : text      # unique within the holder; also the grid's row key
category     : text      # the grid's grouping header
description  : text      # the help text shown when the row is selected
default      : <type>    # what "revert" restores
binding      : either (getter, setter) or a direct reference to the field
```

and four independent behaviour flags, each expressed as a two-valued enumeration rather than a boolean so the call sites read as words:

| Flag | Choices | Meaning |
|---|---|---|
| read-only | read-only / read-write | whether the cell accepts edits |
| notify parent | notify / do not notify | whether editing this marks the containing object changed |
| password | masked / plain | whether the text is hidden as it is typed |
| refresh grid | refresh / do not refresh | whether editing this rebuilds the whole grid, because it changed what other rows should be |

**Notes** — "Refresh the grid on change" is the flag that matters. Some fields determine the *shape* of the object — change a weather effect's type and a different set of fields becomes relevant — and without it the grid would show stale rows. It is set on very few properties and each one is a place where the data model is really a tagged union.

The two-valued enumerations instead of booleans are a readability device at the call site, where six positional arguments in a row would otherwise all be `true`/`false`. A rebuild with named arguments gets the same effect for free.

### The type ladder

| Value type | Editor offered |
|---|---|
| boolean | a checkbox, or a two-item dropdown when the caller supplies the two labels |
| integer | a spin box; optionally clamped to a range; or a dropdown over (value, label) pairs; or a dropdown over labels indexed by the value |
| real | a spin box; optionally clamped to a range; or a dropdown over (value, label) pairs |
| text | a text field; or a **file picker** given an extension, a file mask, a starting folder and a caption; or a dropdown or tree over a supplied list of choices |
| colour | a colour picker |
| three-component vector | three linked spin boxes |
| nested object | a sub-grid over another holder |
| collection | an editable list of holders, with add, remove and reorder |

Two further axes multiply through most of the ladder:

- **Binding style**: a getter/setter pair, or a direct reference to the field. The direct form is the common one and exists to avoid writing two trivial callables per field; the callable form is used when reading or writing needs to do work (recompute a derived value, mark a file dirty).
- **Choice-list style**: a fixed array supplied now, or a pair of callables that supply the array on demand. The deferred form is for lists that change while the editor runs — the set of weather cycles, the set of particle effects — and it is the same pull-not-push decision the [timeline](ide.hpp.md) makes.

The file-picker variant deserves its own note: it is the only property type that carries *four* presentation arguments (default extension, file mask, starting folder, caption) plus two behaviour choices — whether the author may type a path the picker did not offer, and whether the stored value keeps or drops the extension. Dropping the extension is the usual setting, because the engine's resource names are extensionless logical paths and the picker shows real files.

## `property_value`

**Contract** — the handle returned by adding a property. Its only capability is changing one of the four behaviour flags after the fact, so that a property can become read-only when another property changes.

## `property_holder_collection`

**Contract** — an editable list of holders, implemented by the engine side and driven by the grid.

```text
FUNCTION size() -> int
FUNCTION item(position) -> PropertyHolder
FUNCTION display_name(position, out buffer, buffer_size)
FUNCTION index_of(holder) -> int
FUNCTION create() -> PropertyHolder       # the grid's "add" button
FUNCTION insert(holder, position)
FUNCTION erase(position)
FUNCTION destroy(holder)
FUNCTION clear()
```

**Invariants** — `create` makes an element but does not insert it; the grid inserts it where the author asked. `erase` removes without destroying, so a reorder is an erase and an insert; `destroy` is the actual release. **Separating removal from destruction is what makes drag-reordering safe** and is the collection's one non-obvious rule.

The display name is written into a caller-supplied buffer rather than returned, because a string cannot cross the boundary by value.

## `property_holder_holder`

**Contract** — a one-method interface answering "which holder represents you". It lets a nested object be reached from its containing grid row. Its existence, rather than a direct reference, is what allows an object to *be* editable without *being* a holder — the engine's weather keyframe is an engine type that hands back a holder, not a holder itself.

## Notes

The whole surface is a binding language, and the observation worth carrying into a rebuild is that it is almost exactly a **schema**: identifier, category, description, type, default, constraints, presentation hint, and an accessor pair. Written as data rather than as sixty overloads, it would be a fraction of the size and would also be serializable — which would let the editor show the same grid for an object the engine described once, rather than requiring the engine to be running to populate it.

The overload explosion is genuinely incidental: it is the only way to express an optional-typed-argument matrix in this language without variants. A rebuild should write one add-property call taking a described property record.
