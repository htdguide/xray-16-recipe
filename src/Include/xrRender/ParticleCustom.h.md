# src/Include/xrRender/ParticleCustom.h

> A running particle effect as its owner sees it: play, stop, follow this transform, is it done.

**Needs** — [`RenderVisual.h`](RenderVisual.h.md) · [`xrParticles/README.md`](../../xrParticles/README.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`RenderVisual.h`](RenderVisual.h.md) · [`ParticleEffect.cpp`](../../Layers/xrRender/ParticleEffect.cpp.md) · [`ParticleEffect.h`](../../Layers/xrRender/ParticleEffect.h.md) · [`ParticleGroup.cpp`](../../Layers/xrRender/ParticleGroup.cpp.md) · [`ParticleGroup.h`](../../Layers/xrRender/ParticleGroup.h.md) · [`dxParticleCustom.h`](../../Layers/xrRender/dxParticleCustom.h.md)
**Tier floor** — T2: a control surface over a simulation the renderer owns; nothing crosses it but scalars and a transform.

## Purpose

A particle effect is a [render visual](RenderVisual.h.md) like a mesh is — the engine creates it by name through the renderer's model factory, positions it, and hands it back to be drawn. But unlike a mesh it is *running*: it has a lifetime, it follows a moving emitter, and it finishes. This interface is the extra surface an owner gets by downcasting a visual to a particle effect.

Every muzzle flash, explosion, blood spray, campfire and footstep dust in the game is one of these.

## State

The owner sees only what the calls below expose; the simulation state is the renderer's.

```text
RECORD ParticleEffect         # what an implementor must track
  definition   : ParticleGroupDef     # the authored effect, resolved at create time
  playing      : bool
  stop_pending : bool                 # stopped, but still emitting-out
  parent_xform : Matrix
  parent_vel   : Vector
  hud_mode     : bool
```

## `IParticleCustom` — what an implementor must provide

### `play` / `stop(deferred)` / `is_playing`

**Contract** — `play` starts or restarts emission. `stop` has two modes and the distinction is the most important thing in this file: **deferred stop ends emission but lets the already-emitted particles live out their lifetimes; immediate stop kills them all at once.** Deferred is the default, and it is what a muzzle flash or a thruster wants — cutting particles mid-flight is instantly recognizable as wrong. Immediate is for teleports and for tearing down a level.

`is_playing` stays true through a deferred stop until the last particle dies, which is how an owner knows when the effect may be destroyed.

### `update_parent(transform, velocity, carry)`

**Contract** — tells the effect where its emitter is and how fast it is moving, once per frame while the owner moves. `carry` decides whether already-emitted particles move with the emitter or are left behind in world space. Emitter velocity is separate from the transform because particles inherit it at birth — smoke from a moving vehicle trails correctly only if the emitter's velocity is known, not merely its successive positions.

**Notes** — The `carry` flag is a per-effect authoring decision made at the call site rather than in the effect's definition, which means the same effect behaves differently depending on who plays it. That is a small design wart; a rebuild should consider moving it into the definition.

### `on_frame(dt_ms)`

**Contract** — advances the simulation by an elapsed time in milliseconds. Called by the owner, not by the renderer — the engine drives particle simulation from the game loop so that a paused game has frozen particles.

### `time_limit` / `is_looped`

**Contract** — the effect's authored duration in seconds. **A negative duration means the effect loops forever**, and the looped query is literally that test. This sentinel is baked into shipped effect definitions.

### `particle_count`

**Contract** — how many particles are currently alive. Used for debug statistics and by the engine's budget enforcement, which stops spawning new effects when the total gets large.

### `name`

**Contract** — the effect definition's authored name. Needed because an owner often creates an effect from a name held in configuration and later wants to know what it got.

### `set_hud_mode` / `hud_mode`

**Contract** — whether the effect belongs to the first-person weapon layer. A heads-up effect is drawn with the weapon's own narrow field of view and near plane, after the world, so a muzzle flash does not clip through geometry the weapon is already intersecting.

**Notes** — This flag exists because the first-person weapon is rendered in a *different projection* from the world and must be, or it intersects everything the player stands next to. Any rebuild will meet the same problem and needs the same distinction somewhere; putting it on the effect rather than on the draw call is the choice made here, and it means the owner must remember to set it.

### `on_device_create` / `on_device_destroy`

**Contract** — acquire and release the effect's device resources around a device reset. The simulation state survives; only the buffers and materials are rebuilt.
