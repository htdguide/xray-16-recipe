# src/editors/xrWeatherEditor/property_integer_values_value_reference_getter.cpp

> The live-list index row, with the index aliased onto an engine field.

**Needs** — [`property_integer_values_value_reference_getter.hpp`](property_integer_values_value_reference_getter.hpp.md) · [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_integer_values_value_reference_getter.hpp`](property_integer_values_value_reference_getter.hpp.md); callers name that, not this file.
**Tier floor** — T2: owns native callback objects and an alias into engine-owned storage

## Purpose

The reference-bound twin of [`property_integer_values_value_getter.cpp`](property_integer_values_value_getter.cpp.md). The list is still produced by callbacks — a list has no field to alias — while the index itself is poked directly.

## State

```text
RECORD LiveIndexSelectionPropertyByReference EXTENDS IntegerPropertyByReference
  list_getter : callback() -> native array of text   # owned copy
  list_size   : callback() -> int                    # owned copy
```

## `construct(field, list_getter, list_size)` · `release` · `collection` · `GetValue` · `SetValue`

**Contract** — Identical to the accessor-bound twin in every respect except that the index is read and written through the alias. Both list callbacks are freed exactly once.

**Notes** — This is the one combination in the directory that mixes the two binding flavours in a single adapter, and it shows the flavours are a property of each bound *quantity*, not of the row. A rebuild that makes binding a field rather than a base class gets this combination for free instead of as a fourth class.
