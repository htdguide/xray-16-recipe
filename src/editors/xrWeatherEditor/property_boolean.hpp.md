# src/editors/xrWeatherEditor/property_boolean.hpp

> Declares the callable-bound boolean property.

**Needs** — [`property_boolean.cpp`](property_boolean.cpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_boolean.cpp`](property_boolean.cpp.md) · [`property_boolean_values_value.hpp`](property_boolean_values_value.hpp.md) · [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md)
**Tier floor** — T1: declares a managed type holding two unmanaged callables.

## Purpose

The surface implemented in [`property_boolean.cpp`](property_boolean.cpp.md).

## The exported units

- **`property_boolean`** — a grid binding over a boolean, holding a getter and a setter. Implements the grid's value interface, and is the base of [`property_boolean_values_value`](property_boolean_values_value.hpp.md).
