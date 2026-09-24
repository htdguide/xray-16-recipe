# src/xrGame/script_particles.h

> Declares the standalone script-owned particle effect: one a script creates, positions, animates along an authored path and stops, with no entity behind it.

**Needs** — [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrEngine/ObjectAnimator.h`](../xrEngine/ObjectAnimator.h.md) · [`script_particles.cpp`](script_particles.cpp.md)
**Used by** — [`base_client_classes_script.cpp`](base_client_classes_script.cpp.md) · [`script_particles.cpp`](script_particles.cpp.md) · [`script_particles_inline.h`](script_particles_inline.h.md) · [`script_particles_script.cpp`](script_particles_script.cpp.md)
**Tier floor** — T2: a handle pair with a two-way ownership link

## Purpose

Declares the surface implemented in [`script_particles.cpp`](script_particles.cpp.md).
Unlike the [particle action channel](script_particle_action.h.md), which is one channel of
an entity's scripted action, this is a free-standing effect the script itself holds: it has
no entity, no action queue and no completion flag, and it lives exactly as long as the
script's handle to it.

Two types are declared, and the split between them is the whole design:

- the **script handle** — what a script holds, carrying the effect's transform;
- the **world-side effect** — a particle effect the engine's scheduler steps, extended with
  an optional path animator.

They point at each other, and the interesting question is what happens when one of them
dies first. That is answered in the implementation twin.

## Exported units — the script handle

- construct from an effect name.
- destroy.
- `play`, `play_at(position)` — start the effect where it is, or at a given place.
- `stop`, `stop_deferred` — cut it dead, or let already-emitted particles finish.
- `is_playing`, `is_looped`.
- `move_to(position, velocity)` — reposition and tell the emitter how fast it is travelling.
- `set_direction(direction)`, `set_orientation(yaw, pitch, roll)` — two ways to aim it.
- `last_position` — where it was last placed.
- `load_path(name)`, `start_path(looped)`, `stop_path`, `pause_path(paused)` — drive it
  along an authored motion path.

## Exported units — the world-side effect

- construct from (owner, effect name).
- `scheduled_update(elapsed)` — advances the path animator and re-derives the effect's
  transform and velocity from it.
- `load_path`, `start_path`, `stop_path`, `pause_path` — the animator half of the handle's
  path methods.
- `internal_delete`, `destroy` — the two ways the world side can end, both of which must
  notify the owner.
- `release_owner` — told by the handle that it is going away first.
