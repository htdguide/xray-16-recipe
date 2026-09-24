# src/Layers/xrRender/ParticleGroup.h

> Declares the authored particle-group timeline record, its runtime, the per-entry item that owns an emitter and its two kinds of child effect, and the chunk identifiers of the on-disk record.

**Needs** — [`ParticleGroup.cpp`](ParticleGroup.cpp.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`Include/xrRender/ParticleCustom.h`](../../Include/xrRender/ParticleCustom.h.md)
**Used by** — [`ModelPool.cpp`](ModelPool.cpp.md) · [`ModelPool.h`](ModelPool.h.md) · [`PSLibrary.cpp`](PSLibrary.cpp.md) · [`PSLibrary.h`](PSLibrary.h.md) · [`ParticleGroup.cpp`](ParticleGroup.cpp.md) · [`r__dsgraph_build.cpp`](r__dsgraph_build.cpp.md)
**Tier floor** — T1: it fixes the record version and the numeric chunk identifiers of a shipped format.

## Purpose

Declares the surface implemented in [`ParticleGroup.cpp`](ParticleGroup.cpp.md). The timeline rule, the child-list parallelism invariant and the load format are on that page.

Exported units:

- `GroupDefinition` — the authored group: name, flags, time limit, and a list of timeline entries.
- `GroupEntry` — one line of the timeline: the effect to play, its start and stop times, the names of its three possible child effects, and the entry flag set (`deferred_stop`, `has_on_play_child`, `enabled`, `on_play_child_rewind`, `has_on_birth_child`, `has_on_dead_child` — bit 3 is unused and reserved).
- `load` / `save` and `load_from_config` / `save_to_config`.
- `ParticleGroup` — the runtime: a renderable visual and a particle-custom visual that plays the timeline as one object.
- `GroupItem` — the per-entry runtime: the entry's emitter, the list of children that follow live particles, and the list of free children that run to completion. Carries `start_related_child`, `stop_related_child`, `start_free_child`, its own `on_frame`, `play`, `stop`, `clear`, `particles_count`, and the device-reset pair.
- `compile`, `on_frame`, `update_parent`, `play`, `stop`, `is_playing`, `particle_count`, `name`, `get_time_limit`, `set_hud_mode` / `get_hud_mode`, `on_device_create` / `on_device_destroy`, and `copy`, which fails by design.
- Format constants: record version 3, and the chunk identifiers listed in the implementation twin.
