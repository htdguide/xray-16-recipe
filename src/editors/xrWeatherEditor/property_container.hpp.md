# src/editors/xrWeatherEditor/property_container.hpp

> Declares the grid-facing face of an editable object, and attaches its renderer.

**Needs** — [`property_container.cpp`](property_container.cpp.md) · [`property_container_converter.hpp`](property_container_converter.hpp.md) · [`property_container_holder.hpp`](property_container_holder.hpp.md) · [`xrSdkControls/Controls/Interfaces/IPropertyContainer.cs`](../xrSdkControls/Controls/Interfaces/IPropertyContainer.cs.md)
**Used by** — [`ide_impl.cpp`](ide_impl.cpp.md) · [`property_collection_base.cpp`](property_collection_base.cpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md) · [`property_color_base.cpp`](property_color_base.cpp.md) · [`property_container.cpp`](property_container.cpp.md) · [`property_container_converter.cpp`](property_container_converter.cpp.md) · [`property_container_holder.hpp`](property_container_holder.hpp.md) · [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md) · [`property_converter_color.cpp`](property_converter_color.cpp.md) · [`property_converter_float_enum.cpp`](property_converter_float_enum.cpp.md) · [`property_converter_integer_enum.cpp`](property_converter_integer_enum.cpp.md) · [`property_converter_integer_values.cpp`](property_converter_integer_values.cpp.md) · [`property_converter_string_values.cpp`](property_converter_string_values.cpp.md) · [`property_converter_string_values.hpp`](property_converter_string_values.hpp.md) · _and 21 more_
**Tier floor** — T1: declares a managed type that owns a pointer to an unmanaged holder and has both a deterministic and a collected release path.

## Purpose

The surface implemented in [`property_container.cpp`](property_container.cpp.md), plus the declaration that binds [`property_container_converter`](property_container_converter.hpp.md) to it — which is what makes a nested container render as an expandable sub-grid rather than as a meaningless object reference.

## The exported units

- **`property_container`** — an editable object's rows. Extends the third-party property bag this grid is built on, and implements [`IPropertyContainer`](../xrSdkControls/Controls/Interfaces/IPropertyContainer.cs.md).
- **`add_property`** — record a row description and its live binding, applying the ordering and uniqueness rules.
- **`property_for`** — the description-to-binding lookup the grid's gestures route through.
- **`holder`** — the unmanaged object this container describes.
- **`container_holder`** — the object that owns this container, used by [`property_converter_color`](property_converter_color.cpp.md) to reach a nested value's real owner.
- **`properties`, `ordered_properties`** — the two views of the row set; the ordered one is what [`property_container_converter`](property_container_converter.cpp.md) sorts by.
- **`clear`** — empty the container for refilling.
