# src/editors/xrWeatherEditor/property_integer_values_value_reference.cpp

> The fixed-list index row, bound by alias instead of by callbacks.

**Needs** — [`property_integer_values_value_reference.hpp`](property_integer_values_value_reference.hpp.md) · [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: copies a native text array into a managed list at construction

## Purpose

The reference-bound twin of [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md).

## State

```text
RECORD IndexSelectionPropertyByReference EXTENDS IntegerPropertyByReference
  labels : list<text>       # invariant: non-empty; ordering is the stored index's meaning
```

## `GetValue` · `SetValue` · `collection`

**Contract** — Identical to the accessor-bound twin: clamp the index into the list's bounds on read, store the picked label's position on write, publish the list for the converter.
