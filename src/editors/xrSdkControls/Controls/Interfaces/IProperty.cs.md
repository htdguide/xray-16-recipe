# src/editors/xrSdkControls/Controls/Interfaces/IProperty.cs

> The one thing the grid needs of a value: read it, write it. Everything else about a property is presentation.

**Needs** — _(none)_
**Used by** — [`IPropertyContainer.cs`](IPropertyContainer.cs.md) · [`PropertyGrid.cs`](../PropertyGrid.cs.md) · [`property_boolean.cpp`](../../../xrWeatherEditor/property_boolean.cpp.md) · [`property_boolean_reference.cpp`](../../../xrWeatherEditor/property_boolean_reference.cpp.md) · [`property_container.cpp`](../../../xrWeatherEditor/property_container.cpp.md)
**Tier floor** — T3: two untyped accessors.

## Purpose

The grid does not hold values; it holds bindings. This is the binding. Every property type in the editor — boolean, integer, real, colour, file name, nested object, collection — is reachable through exactly these two calls, and the difference between them lives entirely in the presentation layer (which cell editor, which converter) rather than here.

Keeping the binding untyped is deliberate: the grid is generic over property types, and the one place where the real type is known is the implementation on the engine side, which casts on the way in and boxes on the way out.

## State

`Stateless.`

## `IProperty`

```text
INTERFACE Property
  FUNCTION get() -> any          # called on every paint of the row
  FUNCTION set(value : any)      # called when the author commits an edit
```

**Contract** — `get` must be cheap and side-effect free: it runs once per row per repaint, and the editor repaints continuously while the weather advances. `set` takes whatever the cell editor produced for this row's declared type and is responsible for narrowing it; a mismatch is a programming error, not an input error, because the cell editor was chosen from the same declaration.

**Notes** — there is no change notification here. The grid learns that a value changed because it repaints and re-reads. That is the same pull-not-push decision the whole editor makes, and it is why editor and engine cannot diverge.
