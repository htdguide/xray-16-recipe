# src/editors/xrWeatherEngine/editor_environment_manager.hpp

> Declares the editable weather system: the engine's own weather system with every authored record replaced by one the grid can edit.

**Needs** — [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md) · [`editor_environment_effects_effect.cpp`](editor_environment_effects_effect.cpp.md) · [`editor_environment_effects_manager.cpp`](editor_environment_effects_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md) · [`editor_environment_suns_gradient.cpp`](editor_environment_suns_gradient.cpp.md) · [`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md) · [`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) · [`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md) · [`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md) · [`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md) · [`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md) · [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) · [`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md) · _and 1 more_
**Tier floor** — T2: it substitutes itself for the engine's weather system at run time.

## Purpose

Declares the surface implemented in
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md). It is the run-time
weather system, specialised: it keeps seven sub-managers — one per authored file the
weather model spans — and overrides the points where the run-time system would build an
authored record, so that an editable one is built instead.

## State

```text
RECORD EditorEnvironment EXTENDS Environment
  suns, levels, effects, sound_channels,
  ambients, thunderbolts, weathers   : sub-manager    # each owns one authored file set
  property_holder                    : PropertyHolder # the root of the grid's tree

  shader_ids         : list<text>    # cached, sorted; built on first request
  particle_ids       : list<text>    # cached, sorted; built on first request
  light_animator_ids : list<text>    # cached, sorted; built on first request
```

**Invariants** — every sub-manager exists from construction to teardown; the three
identifier caches are built once and never invalidated, so a rebuild that lets an author
create particle systems or shaders inside this editor must invalidate them.

## Exported units

- **construct / destruct** — builds every sub-manager, loads, and populates the grid.
- **`load_weathers`** — read every weather cycle, sort each one, select the first.
- **`load` / `load_internal` / `unload`** — the engine's load hooks, redirected.
- **`save`** — write every authored file the editor may change.
- **`create_mixer`** — make the editable keyframe that holds interpolated results.
- **`AppendEnvAmb`, `thunderbolt_description`, `thunderbolt_collection`** — the three
  factory hooks the run-time system calls, redirected to the sub-managers.
- **`shader_ids`, `particle_ids`, `light_animator_ids`** — the three browsable name lists
  the grid's pickers need.
- **accessors** — one per sub-manager.
