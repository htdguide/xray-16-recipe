# src/xrParticles/particle_actions_collection.h

> Declares the thirty-one actions an effect can be built from, each as a record of authored
> parameters implementing the common action interface.

**Needs** — [`particle_actions.h`](particle_actions.h.md) · [`particle_core.h`](particle_core.h.md)
**Used by** — [`particle_actions_collection.cpp`](particle_actions_collection.cpp.md) · [`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md) · [`particle_manager.cpp`](particle_manager.cpp.md)
**Tier floor** — T2: parameter records and virtual dispatch; the frozen part is the wire
format, not these declarations.

## Purpose

Declares the surface implemented in
[`particle_actions_collection.cpp`](particle_actions_collection.cpp.md) (behaviour) and
[`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md) (loading). This
is the *catalogue*: the complete set of verbs available to an effect author, frozen because
the shipped data uses them by number.

Each action's parameters, and exactly what it does to a particle per step, are in the
behaviour twin. Listing them twice would be transcription; what belongs here is the shape of
the catalogue as a whole.

Every action that has a spatial parameter declares it **twice** — once in the emitter's local
space, once in world space. The local copy is what the file holds and is never written after
loading; the world copy is re-derived from it whenever the emitter moves
([`particle_manager.cpp`](particle_manager.cpp.md)). The simulation only ever reads the world
copy. A rebuild that keeps one copy and a matrix pays a transform per particle instead of per
emitter move, which is the wrong trade at several hundred particles per effect.

Exported units, in declaration order. The *kind codes* they are stored under are a different
order and are listed in [`psystem.h`](psystem.h.md); the two must not be conflated.

| Action | Role |
|---|---|
| `Avoid` | steer velocity away from a domain before reaching it |
| `Bounce` | reflect velocity off a domain's surface, with friction and restitution |
| `CopyVertexB` | latch each particle's current position into its second position slot |
| `Damping` | scale velocity toward zero, per axis, within a speed band |
| `Explosion` | push particles outward with a Gaussian shell expanding from a centre |
| `Follow` | accelerate each particle toward the next one in the pool |
| `Gravitate` | pairwise attraction among all particles |
| `Gravity` | constant acceleration in one direction |
| `Jet` | random acceleration sampled from a domain, falling off from a centre |
| `KillOld` | retire particles on one side of an age threshold; publishes the lifetime |
| `MatchVelocity` | pairwise velocity exchange among nearby particles |
| `Move` | integrate position from velocity and advance age — the only action that does |
| `OrbitLine` | accelerate toward the nearest point on a line |
| `OrbitPoint` | accelerate toward a point |
| `RandomAccel` | add an acceleration drawn from a domain |
| `RandomDisplace` | add a displacement drawn from a domain |
| `RandomVelocity` | replace velocity with one drawn from a domain |
| `Restore` | steer every particle back to its second position within a countdown |
| `Scatter` | accelerate radially away from a centre |
| `Sink` | retire particles by position against a domain |
| `SinkVelocity` | retire particles by velocity against a domain |
| `Source` | emit new particles at a rate, sampling six domains for their initial state |
| `SpeedLimit` | clamp speed into a band |
| `TargetColor` | ease colour and alpha toward a target over an age window |
| `TargetSize` | ease size toward a target, per axis |
| `TargetRotate` | ease spin magnitude toward a target, preserving direction |
| `TargetVelocity` | ease velocity toward a target |
| `Vortex` | rotate positions about an axis through a centre |
| `Turbulence` | perturb velocity direction by a fractal noise gradient, preserving speed |

**Notes** — the catalogue descends from a published open-source particle API of the late
1990s, which is why several actions solve problems no shipped effect has (pairwise
gravitation among hundreds of particles is quadratic and would not survive a frame budget) and
why the parameter vocabulary — magnitude, epsilon, max radius — repeats across unrelated
actions. A rebuild is not free to prune the catalogue: an unknown kind code is a hard failure
in the loader, so every code that appears in shipped data must be implemented, including the
ones whose effect is subtle enough that nobody would miss them.
