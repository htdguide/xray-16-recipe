# src/editors/xrWeatherEditor/property_editor_color.hpp

> Declares the colour row's swatch painter and modal picker.

**Needs** — [`property_color_base.hpp`](property_color_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`property_editor_color.cpp`](property_editor_color.cpp.md) · [`property_holder_color.cpp`](property_holder_color.cpp.md)
**Tier floor** — T3: pure presentation; the colour crosses by value through the row's own interface

## Purpose

Declares the value editor implemented in [`property_editor_color.cpp`](property_editor_color.cpp.md).

## Exported units

- **`property_editor_color`** — paints the swatch in the grid row and opens the platform colour picker.
- **`GetPaintValueSupported` / `PaintValue`** — the swatch.
- **`GetEditStyle` / `EditValue`** — the modal picker.
