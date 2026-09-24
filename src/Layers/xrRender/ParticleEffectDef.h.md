# src/Layers/xrRender/ParticleEffectDef.h

> Declares the authored particle-effect definition, its frozen flag bits, its atlas frame layout, and the chunk identifiers of the on-disk record.

**Needs** — [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md) · [`Shader.h`](Shader.h.md) · [`xrParticles/psystem.h`](../../xrParticles/psystem.h.md)
**Used by** — [`ModelPool.h`](ModelPool.h.md) · [`PSLibrary.cpp`](PSLibrary.cpp.md) · [`PSLibrary.h`](PSLibrary.h.md) · [`ParticleEffect.cpp`](ParticleEffect.cpp.md) · [`ParticleEffect.h`](ParticleEffect.h.md) · [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md)
**Tier floor** — T1: it fixes the byte image of the frame layout block and the numeric chunk identifiers of a shipped format.

## Purpose

Declares the surface implemented in [`ParticleEffectDef.cpp`](ParticleEffectDef.cpp.md). The full contract — every field, every flag bit, the frame convention, the collision resolve — is on that page.

Exported units:

- `FrameLayout` — the texture-atlas cut: one frame's normalized size, two reserved reals that must round-trip, frames per row, frame count, playback speed. Carries the atlas lookup that turns a frame index into a pair of corner texture coordinates.
- `EffectDefinition` — the definition record itself: flags, material names, compiled material, frame layout, opaque action bytes, time limit, particle budget, velocity scale, path-alignment default rotation, and the three collision constants.
- The definition flag set (`sprite`, `framed`, `animated`, `random_frame`, `random_playback`, `time_limit`, `align_to_path`, `collision`, `collision_del`, `velocity_scale`, `collision_dyn`, `world_align`, `face_align`, `culling`, `cull_ccw`) — bit positions are frozen by the shipped library.
- `load` / `save` and `load_from_config` / `save_to_config` — the binary and text forms of the same record.
- `execute_animate`, `execute_collision` — the two per-step behaviours the renderer adds on top of the simulator's action list.
- `create_material` / `destroy_material` / `set_name` / `name`.
- The collision and destruction callback shapes an instance may install, and the format constants: record version 1, and the chunk identifiers listed in the implementation twin.
