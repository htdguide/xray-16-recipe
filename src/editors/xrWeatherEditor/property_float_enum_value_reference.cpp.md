# src/editors/xrWeatherEditor/property_float_enum_value_reference.cpp

> The named-magnitude real row, bound by alias instead of by callbacks.

**Needs** — [`property_float_enum_value_reference.hpp`](property_float_enum_value_reference.hpp.md) · [`property_float_reference.hpp`](property_float_reference.hpp.md)
**Used by** — reached through its declarations in [`property_float_enum_value_reference.hpp`](property_float_enum_value_reference.hpp.md); callers name that, not this file.
**Tier floor** — T2: copies a native `(magnitude, label)` array into a managed list at construction

## Purpose

The reference-bound twin of [`property_float_enum_value.cpp`](property_float_enum_value.cpp.md). Same list, same snap-on-read, same label-on-write, same suppressed nudge; the value is aliased rather than fetched.

## State

```text
RECORD RealChoicePropertyByReference EXTENDS RealPropertyByReference
  choices : list<Pair<real, text>>     # invariant: non-empty; entry 0 is the fallback
```

## `GetValue` · `SetValue` · `Increment`

**Contract** — Identical to the accessor-bound twin; consult it for the exact-equality hazard and the fallback rule.

**Notes** — The base's step is set to `0.05` here too, and is dead for the same reason. A rebuild collapsing the binding flavours into one adapter removes this file entirely.
