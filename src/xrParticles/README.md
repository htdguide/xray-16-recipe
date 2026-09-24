# src/xrParticles — particle simulation

Chapter 12 of the build order.

## What this module is responsible for

It advances particles and nothing else. It does not draw them, does not know what a texture
is, does not own a shader, and its one drawing entry point has been empty for the life of the
project. What it owns is a **pool** of particle records and an **action list** that mutates
that pool once per step.

That is the whole design, and it is worth stating plainly because it is unusual: there is no
"particle system" object with an update method full of branches. There is a flat array of
particles, and a straight-line sequence of small typed verbs — emit, accelerate, bounce,
recolour, kill — applied to the whole array in order, every step. An authored effect *is* that
sequence. The engine does not interpret the effect; it replays it.

Everything that makes a particle visible — the sprite, the texture atlas and its frame
animation, alignment to the camera or to the path, collision against the level's geometry —
lives on the caller's side of the boundary, in the renderer's particle layer. The caller gets
a mutable view of the pool between steps and runs its own passes over the same records. That
division is why this chapter can be lifted out of the engine almost intact.

## Where it sits

It rests on very little: three-component vectors and a matrix from the math layer, a shared
random source and a byte reader from the core, and an allocator. The particle record's layout
matters because the renderer walks the pool as raw memory; nothing else here is
device-facing.

The build links it against the engine as well, which is a backwards edge in the chapter order
— chapter 12 reaching into chapter 13. It buys exactly one thing: a global flag saying which
of the three games' data is mounted, consulted at one place in the loader to decide whether an
authored record has a two-field tail. A rebuild passes that in as a loader parameter and the
edge disappears.

Its consumer is the renderer's particle layer (chapter 18), which owns the effect and group
*definitions*, the sprite geometry and the collision pass. Nothing else in the engine talks to
this module.

## The load-bearing ideas

Read these five before the twins; the twins are terse because these are stated here.

**1. A particle is a fixed 64-byte record with no identity.** Position, a second position
whose meaning depends on the action list, velocity, size, one spin angle, a packed colour, an
age, an animation frame and a flag word. A particle is alive because its index is below the
pool's live count. Killing one moves the last live particle into the hole — an explicit,
documented rule that the group layer depends on, which makes indices unstable across a step and
forces every kill loop to run backwards. Full record in
[`psystem.h`](psystem.h.md); pool mechanics in [`particle_effect.cpp`](particle_effect.cpp.md).

**2. An effect is an ordered list of typed actions, and the order is the program.** Thirty-one
kinds, each carrying its authored parameters, each applied to the whole pool per step. They do
not call each other and hold no shared state, with one exception that a rebuild must honour:
the step carries a single mutable scalar, the *lifetime hint*, which the kill-old action
publishes (as its age threshold) and the target-colour action consumes (to turn its fractional
time window into seconds). It starts each step at one second. A colour action placed before
the kill action, or an effect with no kill action, therefore ramps against a one-second
lifetime whatever the real lifetime is. Nothing in the file format records this coupling. The
catalogue is in
[`particle_actions_collection.cpp`](particle_actions_collection.cpp.md); the walk is in
[`particle_manager.cpp`](particle_manager.cpp.md).

**3. Geometry is always a domain.** Eleven kinds — point, line, triangle, plane, box, sphere,
cylinder, cone, blob, disc, rectangle — packed into one record whose fields mean different
things per kind. A domain does two things: draw a uniform sample (how particles are born) and
test containment (how they are killed, bounced and avoided). **The sampling rules are part of
the look.** The sphere's direction comes from normalizing a point drawn in a cube and its
radius is drawn linearly between the inner and outer radius; the cone tapers from its apex at
`p1`; five of the eleven kinds report every point as outside because they have no interior,
which turns a kill action pointed at them into a kill-everything. A rebuild that samples
"correctly" instead of identically will not reproduce the shipped effects. All of it is in
[`particle_core.cpp`](particle_core.cpp.md).

**4. Parameters live in two spaces.** Every action with a position, direction or domain keeps
an authored local-space copy and a derived world-space copy. The local one is never written
after loading; the world one is recomputed whenever the emitter is placed, and only the world
one is read during a step. Actions opt in, via a flag bit, to rotating with the emitter rather
than merely following it — so an authored gravity stays pointing down while an authored
emission cone turns with the muzzle it is attached to. A rebuild that keeps one copy and a
matrix pays a transform per particle instead of per emitter move.

**5. The step is fixed, and it is the caller's.** This module steps whatever duration it is
handed; the caller always hands it **33 ms**, accumulating real frame time against that and
running at most **three** steps in one frame. Beyond three, time is dropped rather than caught
up — the comment is explicit that this only bites below ten frames a second, which is already
unplayable, and the alternative is a stall after a level load. So the simulation is
deterministic in step length and lossy in wall-clock time. Particles are born only by the
source action, whose fractional emission rate is dithered against a random draw so that a rate
of seven per second survives a 33 ms step; they die only by the two kill actions, by the
age threshold, or by a hard stop that discards the pool without notifying anyone.

## Effects, groups, and the authored file

The distinction is not made in this module — it is made one chapter up, in the renderer's
particle layer — but nothing here makes sense without it, and the format is frozen because it
ships with the game.

A **particle effect definition** is one pool plus one action list, plus everything about
drawing it: a name, a shader and texture, a frame-animation descriptor (atlas grid, frame
count, playback speed), a maximum particle count, an optional time limit, collision
parameters, and flags for sprite alignment, culling, random start frame and the rest. Its
simulation half is the compiled action blob described in
[`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md); everything else
in it belongs to the renderer.

A **particle group definition** is a named, ordered set of *references* to effect definitions,
each with a time window, an enabled flag, and up to three child-effect names — one played when
the parent plays, one spawned per particle born, one spawned per particle died. This is how an
explosion is authored: a flash effect, a smoke effect windowed to start later, and a spark
effect whose dying particles each spawn a small trail. The birth and death callbacks this
module exposes exist precisely to serve that layer, and the swap-with-last removal rule is
what lets it keep track of which child belongs to which particle.

Both live in one library file mounted from the game data, and both are also present as loose
files with distinct extensions for the effect and the group. Both carry a version number and
are rejected outright on a mismatch rather than being read leniently.

Mapping an authored effect onto this module, end to end: the renderer creates a pool sized by
the definition's maximum and an empty action list, loads the compiled blob into the list,
places the emitter (deriving every action's world parameters), and then per frame accumulates
time, runs up to three 33 ms steps, and between steps runs its own frame-animation and
collision passes over the same pool. Playing the effect clears the source action's silence
flag and rewinds the two actions that carry time; stopping it sets the flag and, in the normal
deferred form, lets the particles already alive finish.

## The files

| File | Role |
|---|---|
| [`psystem.h`](psystem.h.md) | The module's whole public surface: the particle record, the frozen domain and action kind codes, the callbacks, the manager interface |
| [`particle_core.h`](particle_core.h.md) | Declares the domain and the normal-distribution sampler |
| [`particle_core.cpp`](particle_core.cpp.md) | The eleven domains: field mapping per kind, sampling rules, containment rules, placement |
| [`particle_actions.h`](particle_actions.h.md) | The action interface and the ordered, lockable action list |
| [`particle_actions.cpp`](particle_actions.cpp.md) | Empty; the header is entirely inline |
| [`particle_actions_collection.h`](particle_actions_collection.h.md) | Declares the thirty-one action records — the catalogue's shape |
| [`particle_actions_collection.cpp`](particle_actions_collection.cpp.md) | What each of the thirty-one actions does to a particle per step |
| [`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md) | The frozen byte layout of an authored action, per kind |
| [`particle_effect.h`](particle_effect.h.md) | Declares the pool |
| [`particle_effect.cpp`](particle_effect.cpp.md) | The pool: live-prefix array, admission, swap-with-last removal, resize |
| [`particle_manager.h`](particle_manager.h.md) | Declares the manager's private handle tables |
| [`particle_manager.cpp`](particle_manager.cpp.md) | Handles, the step's action walk, play/stop, placement, action-list loading |
| [`noise.h`](noise.h.md) | Declares the gradient noise and its fractal sums |
| [`noise.cpp`](noise.cpp.md) | Three-dimensional gradient noise and the fractal sum the turbulence action uses |
| [`stdafx.h`](stdafx.h.md) | Compilation prelude; the one place the engine dependency enters |
| [`stdafx.cpp`](stdafx.cpp.md) | Build artifact, no content |

## What a rebuild should do differently

Three things in this module are defects rather than decisions, and a rebuild that reproduces
them gains nothing:

- The follow action computes its loop bound as the live count minus one in unsigned
  arithmetic, which wraps on an empty pool.
- The source action fills a particle's second position slot from an unassigned value unless
  vertex-B tracking is on.
- The noise table's initializer reseeds a process-wide random generator that the particle
  system's own normal-distribution sampler draws its sign from, so the first turbulent effect
  in a session silently perturbs everything else's randomness. Give the noise table its own
  generator.

Two more are *not* defects to fix, however much they look like it: the orbit actions' additive
denominator and the target-rotate/target-velocity "derivative" kind codes that resolve to the
plain variants. Both are baked into every authored magnitude in the shipped data.
