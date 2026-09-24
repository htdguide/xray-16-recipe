# src/editors/xrWeatherEngine/editor_environment_suns_flare.hpp

> Declares one flare in a sun's series: a texture, an opacity, a position along the line, and a radius.

**Needs** — [`editor_environment_suns_flare.cpp`](editor_environment_suns_flare.cpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_suns_flare.cpp`](editor_environment_suns_flare.cpp.md) · [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md)
**Tier floor** — T3: four values and a file browser.

## Purpose

Declares the surface implemented in
[`editor_environment_suns_flare.cpp`](editor_environment_suns_flare.cpp.md). Reached only
from [`suns_flares`](editor_environment_suns_flares.hpp.md), which is itself unreachable.

## State

```text
RECORD Flare
  texture  : text    # a texture path, extension dropped
  opacity  : real
  position : real    # along the line from the sun through the screen centre
  radius   : real
  property_holder : PropertyHolder
```

## Exported units

- **`fill`** — the flare's four grid rows.
