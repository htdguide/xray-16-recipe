# src/editors/xrWeatherEditor/property_string_values_value_shared_str.cpp

> The fixed-choice text row, bound to an interned-text slot.

**Needs** — [`property_string_values_value_shared_str.hpp`](property_string_values_value_shared_str.hpp.md) · [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_string_values_value_shared_str.hpp`](property_string_values_value_shared_str.hpp.md); callers name that, not this file.
**Tier floor** — T2: copies a native text array into a managed sequence at construction

## Purpose

The interned-text twin of [`property_string_values_value.cpp`](property_string_values_value.cpp.md). This is the combination that carries most of the weather document's asset references: the value is an interned name the engine resolves, and the admissible set is what the artist may choose from.

## State

```text
RECORD RestrictedInternedTextProperty EXTENDS InternedTextProperty
  admissible : queue<text>     # snapshot taken at construction
```

## `construct(engine, slot, values, count)` · `values`

**Contract** — Copies `count` native strings into a managed sequence and publishes it. Reading and writing are the facade-routed ones described in [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md); no validation against the admissible set happens on either.
