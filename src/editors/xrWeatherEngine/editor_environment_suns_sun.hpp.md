# src/editors/xrWeatherEngine/editor_environment_suns_sun.hpp

> Declares one sun record: whether the disc is drawn, how big, and with which material and texture.

**Needs** — [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) · [`xrEngine/xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) · [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md)
**Tier floor** — T2: it is the engine's lens-flare record, extended.

## Purpose

Declares the surface implemented in
[`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md).

## State

```text
RECORD Sun EXTENDS RuntimeLensFlare
  id            : text     # the configuration section name
  use           : bool     # whether the sun disc is drawn at all
  ignore_color  : bool     # whether the disc ignores the keyframe's sun colour
  radius        : real
  shader        : text     # a material-pass name
  texture       : text     # a texture path, extension dropped
  # inherited: the gradient, blend and flare sub-records — NOT read or written here
```

**Invariants** — the six fields above are the only part of the authored record this editor
touches. Everything the base record carries is left at whatever the base loader put there.

## Exported units

- **`load` / `save`** — read and write six fields of the section.
- **`fill`** — the record's six grid rows.
- **`id`** — the record's name.
