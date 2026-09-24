# src/Layers/xrRender/ParticleEffect.h

> Declares the playing particle-effect instance — a renderable visual that fills the particle-custom interface — together with the fixed simulation step the whole particle layer is timed by.

**Needs** — [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`Include/xrRender/ParticleCustom.h`](../../Include/xrRender/ParticleCustom.h.md)
**Used by** — [`ModelPool.cpp`](ModelPool.cpp.md) · [`PSLibrary.cpp`](PSLibrary.cpp.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md) · [`ParticleGroup.cpp`](ParticleGroup.cpp.md) · [`ParticleGroup.h`](ParticleGroup.h.md)
**Tier floor** — T2: a declaration only; it names a device-facing implementation but demands nothing of layout itself.

## Purpose

Declares the surface implemented in [`ParticleEffect.cpp`](ParticleEffect.cpp.md). The contracts, the fixed-step rule and the sprite-orientation algorithm are on that page.

Exported units:

- `ParticleEffect` — one playing instance: it is both a renderable visual (so it is culled and drawn like a mesh) and a particle-custom visual (so the game can play, stop and query it without knowing which renderer is loaded).
- The runtime flag set — `playing`, `deferred_stop`, `transform_mode`, `hud_mode`.
- `on_frame`, `render`, `compile`, `update_parent`, `play`, `stop`, `is_playing`, `particle_count`, `name`, `get_time_limit`, `set_hud_mode` / `get_hud_mode`.
- `on_device_create` / `on_device_destroy`, and `copy`, which fails by design.
- `set_destroy_callback`, `set_collision_callback`, `set_birth_death_callbacks` — the hooks the group layer and the game use to react to individual particles.
- `on_particle_birth`, `on_particle_death` — the default per-particle hooks, applying the definition's random-frame and random-playback choices.
- The fixed simulation step, exported in both milliseconds and seconds because both forms are used at call sites elsewhere in the particle layer.
