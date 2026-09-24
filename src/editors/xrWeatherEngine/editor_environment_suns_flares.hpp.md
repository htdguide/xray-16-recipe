# src/editors/xrWeatherEngine/editor_environment_suns_flares.hpp

> Declares the flare series of a sun: a switch, a shared material, and an ordered list of flares along the line from the sun to the screen centre.

**Needs** — [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md)
**Tier floor** — T2: it owns its flare list.

## Purpose

Declares the surface implemented in
[`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md).

**Unreachable in the shipping editor.** Nothing constructs it: the sun record does not hold
one. It is a complete authoring surface for part of the sun record that
[`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) never wires up.

## State

```text
RECORD Flares
  use        : bool               # whether the flare series is drawn
  shader     : text               # one material pass shared by every flare
  flares     : list<Flare>
  collection : PropertyCollection
```

## Exported units

- **`load`** — read the four parallel lists that encode the series.
- **`fill`** — three grid rows: the switch, the material, the list.
- **`save`** — declared; **never defined**.
