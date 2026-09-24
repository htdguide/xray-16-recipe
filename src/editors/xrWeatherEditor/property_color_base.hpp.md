# src/editors/xrWeatherEditor/property_color_base.hpp

> Declares the composite colour binding, its three-channel accessor helper, and the boundary-safe colour record.

**Needs** — [`property_color_base.cpp`](property_color_base.cpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`property_container_holder.hpp`](property_container_holder.hpp.md) · [`xrSdkControls/Controls/Interfaces/IIncrementable.cs`](../xrSdkControls/Controls/Interfaces/IIncrementable.cs.md) · [`xrSdkControls/Controls/Interfaces/IMouseListener.cs`](../xrSdkControls/Controls/Interfaces/IMouseListener.cs.md)
**Used by** — [`property_color.hpp`](property_color.hpp.md) · [`property_color_base.cpp`](property_color_base.cpp.md) · [`property_color_reference.hpp`](property_color_reference.hpp.md) · [`property_editor_color.cpp`](property_editor_color.cpp.md) · [`property_editor_color.hpp`](property_editor_color.hpp.md) · [`window_view.cpp`](window_view.cpp.md) · [`window_weather_editor.cpp`](window_weather_editor.cpp.md)
**Tier floor** — T1: it declares a value record whose layout must match on both sides of a two-language module boundary.

## Purpose

The surface implemented in [`property_color_base.cpp`](property_color_base.cpp.md).

## The exported units

- **`Color`** — a three-real value record. Declared here, and in [`property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) on the other side of the boundary, because the two modules are separately compiled in different languages and neither may assume the other's layout rules. A rebuild that merges the two halves deletes it.
- **`color_components`** — an unmanaged helper exposing a colour's three channels as six callables, each doing a read-modify-write of the whole colour. It exists because a property description needs unmanaged accessors and the colour lives behind a managed binding.
- **`property_color_base`** — abstract. Implements the grid's value interface, the [drag-scrub](../xrSdkControls/Controls/Interfaces/IIncrementable.cs.md) and [double-click](../xrSdkControls/Controls/Interfaces/IMouseListener.cs.md) capabilities, and [container ownership](property_container_holder.hpp.md). Its two abstract operations — read the colour, write the colour — are answered by [`property_color`](property_color.hpp.md) and [`property_color_reference`](property_color_reference.hpp.md).
