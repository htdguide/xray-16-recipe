# src/editors/xrWeatherEngine/editor_environment_suns_gradient.hpp

> Declares the soft halo drawn around a sun: whether it is drawn, how big, how bright, and with which material and texture.

**Needs** — [`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md)
**Tier floor** — T3: five values and two pickers.

## Purpose

Declares the surface implemented in
[`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md).
**Unreachable in the shipping editor** — nothing constructs it.

Not to be confused with
[`thunderbolts::gradient`](editor_environment_thunderbolts_gradient.hpp.md), which is a
different record with a similar name and is fully wired.

## State

```text
RECORD Gradient
  use     : bool
  opacity : real
  radius  : real
  shader  : text    # a material-pass name
  texture : text    # a texture path, extension dropped
```

## Exported units

- **`load`** — read the five values with their defaults.
- **`fill`** — five grid rows.
- **`save`** — declared; **never defined**.
