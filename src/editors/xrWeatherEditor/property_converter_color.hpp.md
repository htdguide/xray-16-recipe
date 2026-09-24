# src/editors/xrWeatherEditor/property_converter_color.hpp

> Declares the colour row's text rendering and sub-row ordering.

**Needs** — [`property_converter_color.cpp`](property_converter_color.cpp.md)
**Used by** — [`property_color_base.cpp`](property_color_base.cpp.md) · [`property_converter_color.cpp`](property_converter_color.cpp.md) · [`property_holder_color.cpp`](property_holder_color.cpp.md)
**Tier floor** — T3: declares six overrides.

## Purpose

The surface implemented in [`property_converter_color.cpp`](property_converter_color.cpp.md).

## The exported units

- **`property_converter_color`** — supplies a colour row's three sub-rows in red/green/blue order, renders the colour as three space-separated reals, converts it to the boundary-safe colour record for a cell editor, and parses the text form back.
