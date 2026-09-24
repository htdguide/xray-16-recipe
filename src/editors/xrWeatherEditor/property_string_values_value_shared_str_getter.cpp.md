# src/editors/xrWeatherEditor/property_string_values_value_shared_str_getter.cpp

> The live-choice text row, bound to an interned-text slot.

**Needs** — [`property_string_values_value_shared_str_getter.hpp`](property_string_values_value_shared_str_getter.hpp.md) · [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: owns native callback objects and aliases an engine-owned interned-text handle

## Purpose

The fourth corner of the text grid: value bound by interned slot, admissible set produced live. Completes the two-by-two of binding flavour against list flavour that [`property_holder_string.cpp`](property_holder_string.cpp.md) dispatches into.

## State

```text
RECORD LiveRestrictedInternedTextProperty EXTENDS InternedTextProperty
  list_getter : callback() -> native array of text   # owned copy
  list_size   : callback() -> int                    # owned copy
```

## `construct(engine, slot, list_getter, list_size)` · `release` · `values`

**Contract** — As in [`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md) for the list, and as in [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md) for the value. Both list callbacks are freed exactly once; the aliased slot is not owned.
