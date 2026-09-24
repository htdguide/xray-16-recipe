# src/editors/xrWeatherEngine/editor_environment_thunderbolts_gradient.hpp

> Declares one glow on a thunderbolt: a material, a texture, an opacity and a radius range.

**Needs** — [`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md) · [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md) · [`editor_environment_thunderbolts_thunderbolt.hpp`](editor_environment_thunderbolts_thunderbolt.hpp.md)
**Tier floor** — T1: writing its material or texture re-creates a device resource.

## Purpose

Declares the surface implemented in
[`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md).
It is the engine's flare sub-record, extended — and unlike its similarly-named neighbour
[`suns::gradient`](editor_environment_suns_gradient.hpp.md), it is fully wired and saved.

## State

```text
RECORD Gradient EXTENDS RuntimeThunderboltFlare
  shader  : text   # a material-pass name
  texture : text   # a texture path, extension dropped
  opacity : real
  radius  : (min : real, max : real)
  property_holder : PropertyHolder
```

## Exported units

- **`load` / `save`** — read and write four keys under a caller-supplied prefix.
- **`fill`** — a nested page of five rows under a caller-supplied name.
