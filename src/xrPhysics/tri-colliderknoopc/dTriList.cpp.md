# src/xrPhysics/tri-colliderknoopc/dTriList.cpp

> Registers the level's static geometry as a single collision shape, and routes the three
> primitives that may hit it.

**Needs** — [`dTriList.h`](dTriList.h.md) · [`dxTriList.h`](dxTriList.h.md) · [`dcTriListCollider.h`](dcTriListCollider.h.md) · [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md) · [`dTriCollideK.h`](dTriCollideK.h.md) · [`../ExtendedGeom.h`](../ExtendedGeom.h.md) · [Seam: Rigid-body dynamics](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`dTriList.h`](dTriList.h.md) · [`dxTriList.h`](dxTriList.h.md)
**Tier floor** — T1: a user-registered shape kind with a function table, a declared payload
size and a destructor.

## Purpose

This is where the engine's own collision database is presented to the dynamics library as
though it were an ordinary collision shape. Everything downstream — the broad phase, the
narrow phase, contact generation, the solver — then treats the whole level as one object and
needs to know nothing about trees, triangles or materials.

Three decisions live here and each is worth carrying over.

## State

```text
# one module-level slot holding the shape-kind identifier, assigned on first use
```

## the infinite extent

**Contract** — the mesh shape reports an **infinite** extent and answers "yes" to every
extent-overlap test.

**Notes** — this is the decision that shapes the whole design. The broad phase is therefore
useless against the level: every shape in the world is a broad-phase candidate against it,
every step. The work of culling is moved entirely into the narrow phase, where the collider
queries the collision database with the *moving shape's* box
([`dcTriListCollider.cpp`](dcTriListCollider.cpp.md)) and caches the answer across steps
([`dSortTriPrimitive.h`](dSortTriPrimitive.h.md)).

That is the right trade for this engine and a rebuilder should understand why before changing
it. A level's geometry does not move, so a broad-phase structure over it would be rebuilt
never and queried constantly — which is precisely what the collision database already is. Two
spatial structures over the same immobile triangles would be one too many.

## the static-collision opt-out

**Contract** — before any of the three cases runs, the *other* shape's user data is consulted
for a flag saying whether it collides with static geometry at all. If not, the pair produces
nothing.

**Invariants** — the flag lives on the colliding shape, not on the mesh, and is the mechanism
behind the `ignore_static` spawn option and the shell-level switch in
[`../PhysicsShell.h`](../PhysicsShell.h.md).

**Notes** — the original preserves, commented out, an earlier rule: collide only if at least
one of the two bodies is awake. That was replaced by the explicit flag, and the replacement is
better — a sleeping body that is about to be woken by something else must still have its
contacts against the world, or it sinks on the frame it wakes.

## the three cases

**Contract** — box, sphere and cylinder each check the opt-out and then delegate to the
collider's corresponding entry point. Nothing else may hit the mesh; any other shape kind gets
no collision function and silently passes through the level.

**Notes** — "silently passes through the level" is the failure mode for any new primitive a
rebuild adds, and it does not announce itself. It is worth an explicit refusal.

## creation and destruction

**Contract** — the first mesh shape created registers the shape kind, declaring the payload
size, the collision-function lookup, the infinite extent routine, the always-true
extent-overlap test, and a destructor. Creation then allocates the shape, adds it to a
collision space, stores the two callbacks and constructs the collider that the shape owns.
Destruction releases that collider.

**Notes** — the collider is the only owned resource, and it is where the per-step triangle
cache lives; releasing the mesh shape at level unload is what frees it.

The implementation file for the collider is *included into this one* rather than compiled
separately, so that its three entry points can be inlined into these three routines. That is
purely a compilation decision and has no successor in a rebuild; it does mean that
[`dcTriListCollider.cpp`](dcTriListCollider.cpp.md) is not a translation unit of its own.
