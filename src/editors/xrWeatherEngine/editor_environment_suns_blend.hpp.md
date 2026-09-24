# src/editors/xrWeatherEngine/editor_environment_suns_blend.hpp

> Declares how a sun's flare fades in and out as the sun enters and leaves view.

**Needs** — [`editor_environment_suns_blend.cpp`](editor_environment_suns_blend.cpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_suns_blend.cpp`](editor_environment_suns_blend.cpp.md)
**Tier floor** — T3: three numbers and three grid rows.

## Purpose

Declares the surface implemented in
[`editor_environment_suns_blend.cpp`](editor_environment_suns_blend.cpp.md).
**Unreachable in the shipping editor** — nothing constructs it.

## State

```text
RECORD Blend
  rise_time : real    # how long the flare takes to appear
  down_time : real    # how long it takes to disappear
  time      : real    # the blend's own step
```

## Exported units

- **`load`** — read the three values with their defaults.
- **`fill`** — three grid rows.
- **`save`** — declared; **never defined**.
