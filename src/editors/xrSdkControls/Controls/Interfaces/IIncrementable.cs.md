# src/editors/xrSdkControls/Controls/Interfaces/IIncrementable.cs

> An optional capability a property may claim: "you may drag me".

**Needs** — _(none)_
**Used by** — [`PropertyGrid.cs`](../PropertyGrid.cs.md) · [`property_color_base.cpp`](../../../xrWeatherEditor/property_color_base.cpp.md) · [`property_color_base.hpp`](../../../xrWeatherEditor/property_color_base.hpp.md)
**Tier floor** — T3: one method.

## Purpose

Middle-dragging across a grid row scrubs its value. Only some properties can be scrubbed — a real, an integer, a colour's brightness — and the grid must be able to ask. This is the question, phrased as a capability the property either claims or does not.

## State

`Stateless.`

## `IIncrementable`

```text
INTERFACE Incrementable
  FUNCTION increment(pixels : real)
```

**Contract** — the argument is a *horizontal mouse displacement in pixels*, positive to the right. The property converts pixels to its own units and applies the change immediately, clamping to its own range. It is called many times per second while a drag is in progress, so it must be cheap and must not allocate a dialog, a file handle, or anything else with a lifetime.

**Notes** — expressing the gesture in pixels and letting each property choose its own sensitivity is the decision worth keeping. The alternative — a normalized fraction of the property's range — makes a 0-to-1 fog density and a 0-to-100000 view distance feel identical to drag, which is exactly wrong for authoring.
