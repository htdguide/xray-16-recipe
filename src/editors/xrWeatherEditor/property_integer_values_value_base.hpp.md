# src/editors/xrWeatherEditor/property_integer_values_value_base.hpp

> The contract a converter relies on to ask a whole-number row for the label list it indexes into.

**Needs** — _(none)_
**Used by** — [`property_converter_integer_values.cpp`](property_converter_integer_values.cpp.md) · [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md) · [`property_integer_values_value.hpp`](property_integer_values_value.hpp.md) · [`property_integer_values_value_getter.cpp`](property_integer_values_value_getter.cpp.md) · [`property_integer_values_value_getter.hpp`](property_integer_values_value_getter.hpp.md) · [`property_integer_values_value_reference.cpp`](property_integer_values_value_reference.cpp.md) · [`property_integer_values_value_reference.hpp`](property_integer_values_value_reference.hpp.md) · [`property_integer_values_value_reference_getter.cpp`](property_integer_values_value_reference_getter.cpp.md) · [`property_integer_values_value_reference_getter.hpp`](property_integer_values_value_reference_getter.hpp.md)
**Tier floor** — T3: a pure presentation-side capability declaration

## Purpose

An interface with one question on it: *what are the labels?* The four index-selection adapters answer it, and the converter that renders and parses those rows asks it. It exists so the converter can serve all four without knowing whether the list was snapshotted at registration or is regenerated per query, and without knowing how the underlying number is bound.

This is a substantive interface, not a forwarding header: what it demands of an implementor is the whole contract.

## State

Stateless.

## `collection`

**Contract** — Returns the ordered label list this property indexes into, as a positional sequence. Position is meaning: the number stored in the document is an index into exactly this sequence. May be recomputed on every call and may therefore return a different list than it did a moment ago; callers must not cache it across a user interaction. Returns a list that is never empty in practice, and no implementor defines behaviour for an empty one.

```text
FUNCTION collection() -> list<text>
```

**Invariants** — The order returned is the order the stored index refers to. An implementation that sorts or filters the list breaks every document written against it.
