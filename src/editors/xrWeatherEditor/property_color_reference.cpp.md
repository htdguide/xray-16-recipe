# src/editors/xrWeatherEditor/property_color_reference.cpp

> The colour binding, reading and writing a colour field directly.

**Needs** — [`property_color_reference.hpp`](property_color_reference.hpp.md) · [`property_color_base.cpp`](property_color_base.cpp.md)
**Used by** — [`property_color_reference.hpp`](property_color_reference.hpp.md)
**Tier floor** — T1: it holds a reference to an unmanaged field for a lifetime it does not control.

## Purpose

The reference half of the colour pair, and the common one: almost every colour in a weather keyframe is a plain field that needs no work on read or write, so binding the field rather than writing two trivial callables is what the engine side actually does.

## State

```text
RECORD FieldColorBinding EXTENDS ColorBinding
  target : reference to Color       # not owned
```

**Invariants** — the referenced colour must outlive the binding; upheld by the holder/object lifetime discipline, not by the type.

## Notes

As in [`property_color`](property_color.cpp.md), the base class is constructed with the current colour so the component rows get their defaults — here that is just the referenced field's present value, so the initialisation-order trap is milder but still present.
