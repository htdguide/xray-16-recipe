# src/editors/xrWeatherEditor/property_color.cpp

> The colour binding, reading and writing through the engine's callables.

**Needs** — [`property_color.hpp`](property_color.hpp.md) · [`property_color_base.cpp`](property_color_base.cpp.md)
**Used by** — [`property_color.hpp`](property_color.hpp.md)
**Tier floor** — T1: it owns two unmanaged callables from a managed object.

## Purpose

The callable half of the colour pair. It answers [`property_color_base`](property_color_base.cpp.md)'s two abstract operations with the getter and setter the engine supplied.

## State

```text
RECORD CallableColorBinding EXTENDS ColorBinding
  get : callable() -> Color        # owned copies
  set : callable(Color)
```

**Invariants** — the callables are copied at construction, released once on either disposal path — the same ownership rule as every callable-bound property in this directory.

## Notes

One subtlety: the base class needs the *current* colour during its own construction, to seed the three component rows' default values. So the getter is invoked once before the binding is fully built, and the result is handed to the base.

That makes the initialisation order load-bearing — the callables must be stored before the base is constructed with their result. In this language the order is a declaration-order rule rather than the written order, which is a trap; a rebuild that constructs explicitly gets it for free, but must still call the getter before seeding the defaults.
