# src/editors/xrWeatherEditor/property_converter_float.hpp

> Declares the real-number text rendering.

**Needs** — [`property_converter_float.cpp`](property_converter_float.cpp.md)
**Used by** — [`property_color_base.cpp`](property_color_base.cpp.md) · [`property_converter_color.cpp`](property_converter_color.cpp.md) · [`property_converter_float.cpp`](property_converter_float.cpp.md) · [`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md) · [`property_converter_vec3f.hpp`](property_converter_vec3f.hpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md) · [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md)
**Tier floor** — T3: declares four overrides.

## Purpose

The surface implemented in [`property_converter_float.cpp`](property_converter_float.cpp.md).

## The exported units

- **`property_converter_float`** — renders a real to fixed three-decimal text and parses text back to a real. Attached by name to every real-valued row the engine describes, and used directly by [`property_converter_color`](property_converter_color.hpp.md) to render a colour's three channels the same way.
