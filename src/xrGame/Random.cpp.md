# src/xrGame/Random.cpp

> Defines the game module's one shared pseudo-random generator instance.

**Needs** — [`Random.hpp`](Random.hpp.md) · [`xrCore/Math/Random32.hpp`](../xrCore/Math/Random32.hpp.md)
**Used by** — reached through its declarations in [`Random.hpp`](Random.hpp.md); callers name that, not this file.
**Tier floor** — T3: one global object

## Purpose

The game layer has a *second* random generator, distinct from the core's. The separation
is the whole content of the file and it is load-bearing: draws made by gameplay — weapon
dispersion, loot rolls, creature decisions, ambient event timing — must come from a stream
that the renderer, the particle system and the sound layer cannot perturb. If they shared
one generator, the number of particles alive in a frame would change which way a bullet
flew, and conformance's determinism requirement would be unmeetable.

The generator is 32-bit and relies on unsigned wraparound, which is a platform assumption
the preface already records.

## State

```text
Random32 : Generator     # process-global, seeded by the core's default seed at startup
```

**Invariants** — seeding and draw order together define the sequence. A rebuild that wants
reproducible gameplay must ensure every gameplay draw goes through this one stream and
that nothing outside gameplay does, and must seed it explicitly at the start of a session
rather than relying on the default.

## `Random32`

**Contract** — the shared gameplay generator. Not thread-safe, and every caller is on the
main simulation thread; a rebuild that moves gameplay work onto workers must either keep
draws on one thread or give each worker its own deterministically derived stream, because
a mutex would make the sequence depend on scheduling.

**Notes** — a process-global mutable is exactly what the preface's service-locator note
warns about. In a rebuild it belongs to the session, created when a game starts and
destroyed with it, which also gives save games somewhere to record its state.
