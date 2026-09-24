# src/editors/xrWeatherEngine/editor_environment_effects_effect.hpp

> Declares one effect record: a particle system, a sound, an offset, a life time and a wind blast.

**Needs** — [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) · [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md)
**Tier floor** — T1: writing its sound name creates and destroys an audio source.

## Purpose

Declares the surface implemented in
[`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md). Like
the keyframe and the ambient, it *is* the engine's record rather than a copy.

## State

```text
RECORD Effect EXTENDS RuntimeAmbientEffect
  id                   : text     # the configuration section name
  sound_name           : text     # shadows the inherited resolved audio source
  # inherited:
  life_time            : int      # milliseconds
  offset               : vec3     # metres, relative to the listener
  particles            : text     # a particle-system name
  wind_gust_factor     : real
  wind_blast_in_time   : real     # seconds, 0..1000
  wind_blast_out_time  : real     # seconds, 0..1000
  wind_blast_strength  : real
  wind_blast_direction : direction  # authored as one angle; the other component is zero
```

## Exported units

- **`load` / `save`** — read and write one configuration section.
- **`fill`** — the record's ten grid rows.
- **`id`** — the record's name.
