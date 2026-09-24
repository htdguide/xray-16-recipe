# src/editors/xrWeatherEditor/property_boolean_values_value_reference.cpp

> The two-label boolean, bound to a field instead of to callables.

**Needs** — [`property_boolean_values_value_reference.hpp`](property_boolean_values_value_reference.hpp.md) · [`property_boolean_reference.cpp`](property_boolean_reference.cpp.md) · [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md)
**Used by** — [`property_boolean_values_value_reference.hpp`](property_boolean_values_value_reference.hpp.md)
**Tier floor** — T1: it holds a reference to an unmanaged field and converts two host strings at construction.

## Purpose

The fourth cell of the boolean binding matrix: choice presentation, field binding. Identical to [`property_boolean_values_value`](property_boolean_values_value.cpp.md) in every respect except which base it extends.

## State

```text
RECORD BooleanChoiceFieldProperty EXTENDS BooleanFieldProperty
  labels : list<text>       # exactly two; position is meaning, first is false
```

## `set`

**Contract** — finds the chosen label's position among the two and writes the corresponding boolean to the field.

## Notes

This file and [`property_boolean_values_value.cpp`](property_boolean_values_value.cpp.md) are the same twenty lines with one word changed. The duplication is forced: the two differ only in their base, and the language cannot express "add this behaviour to either base" without a template the boundary rules forbid.

A rebuild composes instead of inheriting — a choice presentation *wrapping* any boolean binding — and the four types collapse to two plus one wrapper. That collapse applies to the entire directory: see [the README](README.md).
