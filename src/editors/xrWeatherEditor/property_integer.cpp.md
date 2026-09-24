# src/editors/xrWeatherEditor/property_integer.cpp

> One grid row bound to a whole number in the engine by a getter and a setter.

**Needs** — [`property_integer.hpp`](property_integer.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — reached through its declarations in [`property_integer.hpp`](property_integer.hpp.md); callers name that, not this file.
**Tier floor** — T2: owns native callback objects that must be released on a schedule the collector does not choose

## Purpose

The whole-number counterpart of [`property_float.cpp`](property_float.cpp.md), and the base of the clamped, enumerated and index-selection adapters.

## State

```text
RECORD IntegerProperty
  getter : callback() -> int      # native; owned — a private copy, not an alias
  setter : callback(int)          # native; owned
```

No step: whole numbers in this document are selections far more often than quantities, so the adapter does not answer the grid's increment protocol at all. The grid falls back to typing, which is the right affordance for an index.

## `construct(getter, setter)` · `release`

**Contract** — Takes private copies of the callbacks; frees them exactly once, whether release is triggered by the grid or by the collector. The reasoning behind the exactly-once rule is written out in [`property_float.cpp`](property_float.cpp.md).

## `GetValue` · `SetValue`

**Contract** — Call the getter; convert the grid's value to a whole number and call the setter. The width is the engine's — a value outside it is a programming error in the registration, not a user error, because the grid has already type-checked against the declared type.
