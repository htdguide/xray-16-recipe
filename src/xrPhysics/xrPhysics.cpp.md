# src/xrPhysics/xrPhysics.cpp

> Routes every allocation the dynamics library makes through the engine's own allocator, before anything else in the module runs.

**Needs** — [`xrPhysics.h`](xrPhysics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — reached through its declarations in [`xrPhysics.h`](xrPhysics.h.md); callers name that, not this file.
**Tier floor** — T1: hands a foreign library three raw allocation functions and relies on them being installed before any of its state exists.

## Purpose

One decision, made once per process: the dynamics library does not get to use the platform's
allocator. Every body, joint, contact and internal buffer it creates comes out of the engine's
allocator instead.

Three things ride on that. The engine's memory accounting and leak reporting see physics memory,
which is a meaningful fraction of a level's working set. The alignment guarantee the allocator
gives (16 bytes) holds for the structures the solver reads with wide float operations. And the
`pluggable` allocator seam stays whole — building the engine against a different allocator moves
physics with it instead of leaving a second heap behind.

## State

`Stateless.` The installation itself is the only effect.

## Allocator installation

**Contract** — installs three functions on the dynamics library — allocate, reallocate, free — each
forwarding to the engine allocator. Runs **before** any other code in this module and before any
world, body or geom exists; installing them later would leave earlier allocations owned by one heap
and freed by another.

```text
FUNCTION install_physics_allocator()
  dynamics.set_allocator(size            -> engine_allocate(size))
  dynamics.set_reallocator((ptr, old, new) -> engine_reallocate(ptr, new))
  dynamics.set_deallocator((ptr, size)     -> engine_free(ptr))
```

**Notes** — the library passes the *old* size to reallocate and the size to free; the engine
allocator tracks sizes itself and discards both. A rebuild whose allocator needs the size has it
available, which is why the parameters are in the signature at all.

The "before anything else" ordering is expressed in the original as a file-scope object whose
construction runs at module load. That mechanism is incidental; the ordering constraint is not. A
rebuild needs some equivalent of a module-initialization hook, or an explicit call at the very top
of physics start-up before the world is created.
