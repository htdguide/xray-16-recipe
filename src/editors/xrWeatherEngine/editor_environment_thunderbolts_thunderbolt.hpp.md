# src/editors/xrWeatherEngine/editor_environment_thunderbolts_thunderbolt.hpp

> Declares one thunderbolt: a mesh, a sound, a light curve, and the two glowing gradients drawn at its ends.

**Needs** — [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md) · [`editor_environment_thunderbolts_gradient.hpp`](editor_environment_thunderbolts_gradient.hpp.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md)
**Tier floor** — T2: it is the engine's thunderbolt description, extended.

## Purpose

Declares the surface implemented in
[`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md).

## State

```text
RECORD Thunderbolt EXTENDS RuntimeThunderboltDescription
  id              : text      # the configuration section name
  color_animator  : text      # a light-animation curve name
  lighting_model  : text      # a mesh path, extension kept
  sound           : text      # a sound path, extension dropped
  center          : Gradient  # the glow at the bolt's midpoint
  top             : Gradient  # the glow at the bolt's top
```

**Invariants** — both gradients must exist before save or fill; they are created only by
the base loader's callbacks.

## Exported units

- **`load` / `save`** — read and write one configuration section.
- **`fill`** — four rows plus the two gradients' nested pages.
- **`create_top_gradient` / `create_center_gradient`** — the base loader's two callbacks,
  redirected to build editable gradients.
- **`id`** — the record's name.
