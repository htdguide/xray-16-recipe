# src/editors/xrSdkControls/Controls/Interfaces/IPropertyContainer.cs

> How the grid gets from a row it is drawing to the binding behind that row.

**Needs** — [`IProperty.cs`](IProperty.cs.md)
**Used by** — [`PropertyGrid.cs`](../PropertyGrid.cs.md) · [`property_container.cpp`](../../../xrWeatherEditor/property_container.cpp.md) · [`property_container.hpp`](../../../xrWeatherEditor/property_container.hpp.md)
**Tier floor** — T3: one lookup.

## Purpose

The grid widget's own model describes a row by a *specification* — name, category, declared type, description, cell editor. That specification is what the widget hands back on a mouse event. The editor's bindings are stored beside it. This interface is the join.

It exists as its own interface, rather than being a method on the container class, so that the extended [`PropertyGrid`](../PropertyGrid.cs.md) can test any selected object for the capability without knowing the editor's types.

## State

`Stateless.`

## `IPropertyContainer`

```text
INTERFACE PropertyContainer
  FUNCTION property_for(spec : PropertySpec) -> Property
```

**Contract** — the specification must be one this container added; looking up a foreign specification is a programming error. The returned binding is the live one, not a copy.

**Notes** — `PropertySpec` comes from the third-party property-bag library this grid is built on, which is why it appears in an otherwise self-contained interface. A rebuild that writes its own grid uses its own row key and this interface becomes a map lookup.
