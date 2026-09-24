# src/editors/xrWeatherEditor/property_holder_include.hpp

> Declares the compilation boundary between the native engine interface and the managed user-interface layer, and the two tiny adapters every property value in the editor is built from.

**Needs** — [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_boolean.hpp`](property_boolean.hpp.md) · [`property_boolean_reference.cpp`](property_boolean_reference.cpp.md) · [`property_boolean_reference.hpp`](property_boolean_reference.hpp.md) · [`property_collection_base.hpp`](property_collection_base.hpp.md) · [`property_collection_enumerator.hpp`](property_collection_enumerator.hpp.md) · [`property_color_base.hpp`](property_color_base.hpp.md) · [`property_editor_file_name.hpp`](property_editor_file_name.hpp.md) · [`property_editor_tree_values.hpp`](property_editor_tree_values.hpp.md) · [`property_float.cpp`](property_float.cpp.md) · [`property_float.hpp`](property_float.hpp.md) · [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md) · [`property_float_enum_value_reference.hpp`](property_float_enum_value_reference.hpp.md) · [`property_float_reference.cpp`](property_float_reference.cpp.md) · [`property_float_reference.hpp`](property_float_reference.hpp.md) · _and 11 more_
**Tier floor** — T2: names where the foreign-function boundary falls and declares an alias into engine-owned storage; a tier that cannot hold a raw alias to another runtime's field cannot express it

## Purpose

The weather editor is one process with two halves: a managed user-interface layer that owns the window and the property grid, and a native engine that owns the world being edited. This file is where every other file in the property layer states which half it is compiling for, and it supplies the two helpers that let a value living in the native half be edited from the managed half.

It exists as a separate file for one reason: the engine's abstract editor interface must be read by the *native* compiler — it describes callback objects and records with an engine-chosen layout — while everything that follows must be read by the *managed* compiler. Getting that order wrong silently changes the layout of the records crossing the boundary. Centralising the switch means it is stated once.

## State

Stateless as a module. It declares two shapes.

```text
RECORD Pair<first_type, second_type>       # a managed-side two-field tuple
  first  : first_type
  second : second_type

RECORD ValueAlias<T>                        # an alias to storage the engine owns
  target : reference to T                   # invariant: the engine record holding
                                            # target outlives this alias, and nothing
                                            # here enforces that
```

## The boundary declaration

**Contract** — Everything the engine's editor interface declares — the callback pair types, the `color` and `vec3f` records, the flag enumerations — is compiled as native. Everything declared after the switch is compiled as managed. A rebuild that runs the two halves in one process must make the same declaration somewhere; a rebuild that runs them in two processes replaces it with a wire format.

**Notes** — The records that cross (`color` as three reals, `vec3f` as three reals) are packed to a four-byte boundary on the engine side. That packing is not decoration: the managed half reads and writes these records by value, so both halves must agree on the layout byte for byte.

## `ValueAlias`

**Contract** — Wraps a reference to a scalar field the engine owns and exposes read and write. No copy is made: a read returns the engine's current value and a write lands in the engine's field immediately. Not copyable — duplicating an alias would make it ambiguous which copy a later release refers to.

**Invariants** — The aliased field must outlive the alias. The property layer relies on this everywhere and checks it nowhere.

**Notes** — This is the *reference-bound* half of a split that runs through the whole property layer. A property value is bound to the engine one of two ways: by a pair of callbacks the engine supplies (*accessor-bound*), or by a direct alias to one of its fields (*reference-bound*). Accessor binding lets the engine run logic on write — invalidate a cached blend, mark the document dirty; reference binding is a raw poke and is chosen where the field is inert authored data. Every value type in this directory therefore appears twice, once per flavour.

## `Pair`

**Contract** — A managed two-field tuple used to hold an enumerated choice: the numeric value the engine stores, paired with the label the user picks. Exists because the engine hands such lists across as a native array of pairs, and the managed grid needs them as managed objects it can enumerate.
