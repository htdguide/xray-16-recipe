# src/xrParticles/psystem.h

> The module's entire public surface: the particle record, the domain and action type codes,
> the birth/death callbacks, and the handle-based manager interface every caller drives.

**Needs** — [`xrCore/_vector3d.h`](../xrCore/_vector3d.h.md) · [`xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md)
**Used by** — [`ParticleEffect.cpp`](../Layers/xrRender/ParticleEffect.cpp.md) · [`ParticleEffectDef.cpp`](../Layers/xrRender/ParticleEffectDef.cpp.md) · [`ParticleEffectDef.h`](../Layers/xrRender/ParticleEffectDef.h.md) · [`ParticleGroup.cpp`](../Layers/xrRender/ParticleGroup.cpp.md) · [`particle_actions.h`](particle_actions.h.md) · [`particle_actions_collection.cpp`](particle_actions_collection.cpp.md) · [`particle_core.cpp`](particle_core.cpp.md) · [`particle_core.h`](particle_core.h.md) · [`particle_effect.cpp`](particle_effect.cpp.md) · [`particle_effect.h`](particle_effect.h.md) · [`particle_manager.cpp`](particle_manager.cpp.md) · [`particle_manager.h`](particle_manager.h.md) · [`stdafx.h`](stdafx.h.md)
**Tier floor** — T1: the particle record is a fixed 64-byte layout that the renderer walks as
an array and writes into directly, and the type codes are frozen by shipped data files.

## Purpose

This is the only file a caller outside the module reads. It fixes three things that a rebuild
cannot choose freely: the *shape of a particle* (because the renderer iterates the pool in
place and mutates fields of it), the *numbering of the domain and action type codes* (because
authored effect files store them as integers), and the *shape of the manager interface*
(because that interface is the whole module — creation, playback control, one step, and
access to the pool).

Everything else about the module is implementation. The split between this header and the
rest is therefore not arbitrary: this is the frozen part.

## State

```text
# Throughout this chapter, vec3 is shorthand for a triple of reals with the usual
# dot, cross, scale and length operations. Nothing in the chapter needs it to be
# anything more; the engine's own 3-vector is what fills it here.
RECORD vec3
  x, y, z : real
```

```text
# A single particle. This is the record the simulation steps and the renderer draws.
# Exactly 64 bytes with 4-byte fields in the order shown; the renderer indexes the pool
# as a flat array and a rebuild that reorders or widens fields must update it in lockstep.
RECORD Particle
  rot     : real          # one angle, radians, about the view axis. Signed: the sign is
                          # the spin direction and is preserved by the rotate action.
  pos     : vec3          # current world position
  posB    : vec3          # the "other" position. Its meaning depends on the action list:
                          # the previous step's position (move action), the birth position
                          # (source with vertex-B tracking), or the restore target.
                          # The renderer uses it as the tail of a stretched sprite and as
                          # the near end of the collision segment.
  vel     : vec3          # world velocity, units per second
  size    : vec3          # sprite half-extents; only x and y are drawn, z is carried
  color   : int (32-bit)  # packed A,R,G,B, one byte each
  age     : real          # seconds since birth; the only clock a particle has
  frame   : int (16-bit)  # animation frame, stored as frame_index scaled by 255
  flags   : int (16-bit)  # bit 0 = spin/animate counter-clockwise; other bits unused here
```

The record carries no "alive" bit and no identity. A particle is alive because it sits below
the pool's live count; see [`particle_effect.cpp`](particle_effect.cpp.md). Its index is its
identity, and that index is *not* stable across a step — a removal moves the last live
particle into the hole. Any caller that wants to follow one particle (the group layer binds
child effects to individual particles) must track it through the birth and death callbacks
rather than by index.

```text
ENUM DomainKind            # frozen: stored as a 32-bit integer in authored files
  Point = 0        Line = 1       Triangle = 2   Plane = 3      Box = 4
  Sphere = 5       Cylinder = 6   Cone = 7       Blob = 8       Disc = 9
  Rectangle = 10
```

```text
ENUM ActionKind            # frozen: stored as a 32-bit integer in authored files.
                           # Slot 2 is a retired action and must stay a hole.
  Avoid = 0            Bounce = 1           (retired) = 2        CopyVertexB = 3
  Damping = 4          Explosion = 5        Follow = 6           Gravitate = 7
  Gravity = 8          Jet = 9              KillOld = 10         MatchVelocity = 11
  Move = 12            OrbitLine = 13       OrbitPoint = 14      RandomAccel = 15
  RandomDisplace = 16  RandomVelocity = 17  Restore = 18         Sink = 19
  SinkVelocity = 20    Source = 21          SpeedLimit = 22      TargetColor = 23
  TargetSize = 24      TargetRotate = 25    TargetRotateD = 26   TargetVelocity = 27
  TargetVelocityD = 28 Vortex = 29          Turbulence = 30      Scatter = 31
```

Two of those codes are aliases rather than distinct behaviours: `TargetRotateD` and
`TargetVelocityD` construct the same actions as `TargetRotate` and `TargetVelocity` and read
the same payload. The "D" was meant to mark a per-second (rate) variant and the variant was
never written. Shipped data contains both codes, so both must be accepted; a rebuild that
implements the rate semantics for real will change the look of effects that use them.

## Numeric sentinels

```text
CONSTANT UNBOUNDED = 1.0e16     # "no limit"
```

Radius and look-ahead parameters are compared against this value rather than against an
optional: a parameter at or above it selects the branch that skips the range test entirely.
The value is capped well below the square root of the largest representable float because the
comparisons are done on squared quantities and must not overflow. A rebuild is free to use a
real optional instead, as long as authored files that store the huge number still read as
"unbounded".

## `Particle`

**Contract** — see State. Constructed only by the pool's add operation; never allocated
individually.

## `OnBirthParticle` / `OnDeadParticle`

**Contract** — two caller-supplied notifications, each receiving an opaque owner reference, a
caller-chosen integer, the particle itself and its index in the pool. Birth fires after the
particle's fields are filled and *before* the live count is raised, so the index passed is the
index the particle is about to occupy. Death fires before the hole is filled, so the callback
still sees the dying particle's final state. Both are called from inside the step, with the
action list held, and neither may add to or remove from the pool.

These exist so the group layer can spawn a child effect at the position of a particle that was
just born and stop it when that particle dies.

## `DomainKind`, `ActionKind`

**Contract** — frozen integer codes; see State.

## `ParticleManager`

**Contract** — the module's whole interface. It is *handle-based*: effects and action lists
are addressed by small non-negative integers, not by references, so a caller can hold onto one
across a level's lifetime without owning anything. Every call is safe to make from any thread;
the implementation serializes on its own locks. See
[`particle_manager.cpp`](particle_manager.cpp.md) for the contracts of each operation.

```text
INTERFACE ParticleManager
  # lifecycle
  create_effect(max_particles) -> int          # a pool handle
  destroy_effect(effect)
  create_action_list() -> int                  # a list handle
  destroy_action_list(list)

  # playback control: these reach into the list and poke individual actions
  play(effect, list)
  stop(effect, list, deferred)

  # one simulation step of the given duration
  update(effect, list, dt)
  render(effect)                               # present but empty; see notes
  transform(list, placement, parent_velocity)  # re-derive world-space action parameters

  # pool access
  remove_particle(effect, index)
  set_max_particles(effect, n)
  set_callbacks(effect, on_birth, on_dead, owner, param)
  get_particles(effect) -> (pool, live_count)  # direct, mutable view of the live prefix
  get_particle_count(effect) -> int

  # action list contents
  create_action(kind) -> Action
  load_actions(list, reader) -> int            # replaces the list; returns action count
  save_actions(list, writer)
```

**Notes** — `render` is empty and has been for the life of the module. Its presence is the
statement this chapter exists to make: the simulation does not know how particles are drawn.
A rebuild drops it.

`get_particles` hands out a mutable view into the pool so the caller can run *its own* passes
over the same particles — frame animation and world collision both live on the caller's side
(see [`README.md`](README.md)) and mutate `frame`, `pos`, `posB` and `vel` directly between
steps. That aliasing is the reason the pool is a plain array with a live-prefix convention
rather than anything with an ownership discipline.

## `particle_manager` accessor

**Contract** — returns the one process-wide manager. There is exactly one, created before any
caller runs and destroyed with the process. A rebuild that prefers an explicitly constructed
service loses nothing: the manager holds only its two handle tables.
