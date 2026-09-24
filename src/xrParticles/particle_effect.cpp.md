# src/xrParticles/particle_effect.cpp

> The particle pool: a flat array whose live particles are its first *n* entries, with
> swap-with-last removal that the rest of the engine depends on by name.

**Needs** — [`particle_effect.h`](particle_effect.h.md) · [`psystem.h`](psystem.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — reached through its declarations in [`particle_effect.h`](particle_effect.h.md); callers name that, not this file.
**Tier floor** — T1: one contiguous allocation of fixed-layout records, handed out by
reference to the renderer, which walks it as raw memory.

## Purpose

A pool of particles and the two operations that change how many are live. There is no
free list, no generation counter and no identity: a particle is live because its index is
below the live count, and birth and death are a single increment and a single swap.

That choice is what makes the pool cheap enough to iterate thirty-one times per step over
several hundred particles without touching an allocator. It is also what makes indices
unstable, which is the single most important fact about this file.

## State

```text
RECORD ParticleEffect
  live       : int            # particles currently alive
  capacity   : int            # the effect's configured maximum
  allocated  : int            # entries actually allocated
  particles  : list<Particle> # exactly `allocated` entries, one contiguous block
  on_birth   : optional<OnBirthParticle>
  on_dead    : optional<OnDeadParticle>
  owner      : opaque         # passed back to both callbacks
  param      : int (32-bit)   # ditto, a caller-chosen discriminator
```

Invariants:

- `live ≤ capacity ≤ allocated`. Entries at or above `live` are garbage, never read.
- The block is allocated once at construction and only ever grows; shrinking the capacity
  keeps the allocation, so an effect that oscillates between two particle budgets never
  reallocates.
- The block is *not* initialized. A field of a new particle that the admit operation does not
  set holds whatever the previous occupant left there.

## `ParticleEffect(max)` / destructor

**Contract** — allocates a block of `max` particle records, uninitialized, and marks the pool
empty with both callbacks absent. The destructor releases the block. Neither blocks. A
capacity of zero is legal and yields a pool that refuses every admission.

## `resize(n)`

**Contract** — sets the capacity to `n` and returns the capacity actually achieved, which may
be less than asked. Allocates only when `n` exceeds what is already allocated. Never fails
loudly.

```text
FUNCTION resize(pool, n) -> int
  IF n <= pool.allocated THEN
    pool.capacity = n
    # Shrinking below the live count kills the tail outright. The death callback is
    # NOT called for those particles: this is a budget change, not a simulation event.
    IF pool.live > n THEN pool.live = n
    RETURN n

  new_block = allocate(n particles)
  IF new_block is none THEN
    # Out of memory: keep what we have and report the smaller capacity to the caller,
    # who is expected to live with it. The effect keeps running.
    pool.capacity = pool.allocated
    RETURN pool.capacity

  copy the first `live` particles into new_block
  release the old block
  pool.particles = new_block ; pool.capacity = n ; pool.allocated = n
  RETURN n
```

**Notes** — the copy moves only the live prefix, which is the only reason the uninitialized
tail is not a correctness problem. The silent capacity reduction on allocation failure is the
module's entire out-of-memory policy: particles are cosmetic, so degrading the budget is
preferable to failing a level load.

## `remove(index)`

**Contract** — retires the particle at `index`. Calls the death callback first, with the
dying particle and its current index, then fills the hole with the last live particle and
drops the live count. Does nothing at all when the pool is empty — including when `index` is
out of range, which is not checked.

```text
FUNCTION remove(pool, i)
  IF pool.live = 0 THEN RETURN
  IF pool.on_dead THEN pool.on_dead(pool.owner, pool.param, pool.particles[i], i)
  pool.live = pool.live - 1
  pool.particles[i] = pool.particles[pool.live]      # swap the last one into the hole
```

**Invariants** — the fill-from-the-end rule is *not* an implementation detail a rebuild may
change. The group layer binds child effects to individual particles by index and reacts to
the death callback by re-pointing the child that was watching the moved particle. The source
carries an explicit instruction, in Russian, not to change the removal rule because the group
layer depends on it. Any other compaction — a tombstone, a stable free list, an
order-preserving shift — breaks that layer.

**Notes** — the consequence for every caller is that a loop which removes particles must run
*backwards* over the pool, or it will skip the particle that was just moved into the hole.
Every kill action in this chapter does exactly that, and the fact is repeated in each of their
contracts because it is easy to lose in a rebuild.

## `add(pos, posB, size, rot, vel, color, age, frame, flags)`

**Contract** — admits one particle at the end of the live prefix and reports whether it fit.
Refuses, without side effects, when the pool is at capacity. Calls the birth callback after
the fields are filled but *before* the live count is raised, so the index passed is the index
the particle is about to occupy.

```text
FUNCTION add(pool, pos, posB, size, rot, vel, color, age, frame, flags) -> bool
  IF pool.live >= pool.capacity THEN RETURN false
  p = pool.particles[pool.live]
  p.pos = pos ; p.posB = posB ; p.size = size
  p.rot = rot.x                        # only one component of the authored triple survives
  p.vel = vel ; p.color = color ; p.age = age ; p.frame = frame ; p.flags = flags
  IF pool.on_birth THEN pool.on_birth(pool.owner, pool.param, p, pool.live)
  pool.live = pool.live + 1
  RETURN true
```

**Notes** — the caller passes a rotation as a three-component value and only the first
component is kept. The particle record has exactly one angle, about the view axis; the
authoring format still carries a rotation *domain* that samples all three. A rebuild that
sampled a full rotation would be carrying two components nothing reads.
