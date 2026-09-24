# src/editors/xrWeatherEngine/editor_environment_ambients_ambient.hpp

> Declares one ambient record: a name, a firing period, a list of sound channels and a list of effects.

**Needs** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_manager.cpp`](editor_environment_ambients_manager.cpp.md)
**Tier floor** — T2: it is the engine's ambient record, extended.

## Purpose

Declares the surface implemented in
[`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md).
Like the keyframe, it *is* the run-time record rather than a copy of one.

## State

```text
RECORD Ambient EXTENDS RuntimeAmbient
  id                 : text                  # the configuration section name
  effect_period      : (min : int, max : int)  # milliseconds in memory, seconds on disk
  effect_names       : list<EffectId>        # references into the effects file
  sound_channel_names : list<SoundId>        # references into the sound channels file
  # inherited: the resolved effect and sound-channel records the engine plays
```

**Invariants** — the two name lists are the editable view; the inherited resolved lists are
what the engine plays. They are filled in parallel by the same load.

## Exported units

- **`load` / `save`** — read one section across three configurations; write one section.
- **`fill`** — the record's five grid rows.
- **`create_effect` / `create_sound_channel`** — the two factory hooks the base loader
  calls, redirected so the editable child is built.
- **`effects` / `get_snd_channels`** — the engine's accessors, unchanged.
- **`id`**, **`effects_manager`**, **`sounds_manager`**.
