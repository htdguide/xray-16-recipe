# src/editors/xrWeatherEditor/property_integer_reference.cpp

> One grid row aliased straight onto a whole-number field the engine owns.

**Needs** — [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — reached through its declarations in [`property_integer_reference.hpp`](property_integer_reference.hpp.md); callers name that, not this file.
**Tier floor** — T2: holds an alias into another runtime's storage; the alias has no validity check

## Purpose

The reference-bound twin of [`property_integer.cpp`](property_integer.cpp.md).

## State

```text
RECORD IntegerPropertyByReference
  alias : ValueAlias<int>     # aliases a field the engine owns; aliased storage NOT owned
```

## `construct(field)` · `release` · `GetValue` · `SetValue`

**Contract** — Build the alias, free the wrapper exactly once, read and write the aliased field directly. The engine is not told about the write; see [`property_float_reference.cpp`](property_float_reference.cpp.md) for when that is admissible and for the lifetime hazard the alias carries.
