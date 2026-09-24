# src/editors/xrWeatherEditor/property_converter_vec3f.hpp

> Declares the converter that gives a vector row its text form and its three ordered child rows.

**Needs** — [`property_vec3f.hpp`](property_vec3f.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_converter_float.hpp`](property_converter_float.hpp.md)
**Used by** — [`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md) · [`property_holder_vec3f.cpp`](property_holder_vec3f.cpp.md)
**Tier floor** — T3: pure presentation; it reads a vector by value through the row's own interface

## Purpose

Declares the converter implemented in [`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md).

## Exported units

- **`property_converter_vec3f`** — the vector row's text formatter, text parser, and child-row orderer.
- **`GetProperties` / `GetPropertiesSupported`** — the three component rows, in axis order.
- **`CanConvertTo` / `ConvertTo`** — out to text and to the presentation vector.
- **`CanConvertFrom` / `ConvertFrom`** — in from text.
