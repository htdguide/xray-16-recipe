# src/editors/xrWeatherEditor/property_integer_enum_value_reference.cpp

> The named-value whole-number row, bound by alias instead of by callbacks.

**Needs** — [`property_integer_enum_value_reference.hpp`](property_integer_enum_value_reference.hpp.md) · [`property_integer_reference.hpp`](property_integer_reference.hpp.md)
**Used by** — reached through its declarations in [`property_integer_enum_value_reference.hpp`](property_integer_enum_value_reference.hpp.md); callers name that, not this file.
**Tier floor** — T2: copies a native `(value, label)` array into a managed list at construction

## Purpose

The reference-bound twin of [`property_integer_enum_value.cpp`](property_integer_enum_value.cpp.md). Same list, same snap-on-read, same label-on-write.

## State

```text
RECORD IntegerChoicePropertyByReference EXTENDS IntegerPropertyByReference
  choices : list<Pair<int, text>>     # invariant: non-empty; entry 0 is the fallback
```

## `GetValue` · `SetValue`

**Contract** — Identical to the accessor-bound twin, including the quiet rewrite when the document holds an unlisted value.
