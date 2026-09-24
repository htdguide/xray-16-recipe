# src/xrPhysics/tri-colliderknoopc

> The custom triangle-mesh collider: how a rigid body meets the level.

Part of chapter 16, [`src/xrPhysics`](../README.md).

## What this module is responsible for

The level's static geometry is a soup of a few hundred thousand triangles held in the
[static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database).
It is **not** in the solver. The
[dynamics seam](../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) is told about it
through one user-defined shape kind — a single, infinitely large shape standing for "the
level" — and every time a body's box, sphere or cylinder is tested against that shape, the
code in this directory queries the database for nearby triangles and manufactures the
contacts itself.

That arrangement is the chapter's central design decision and it is worth being explicit
about why it was made. Handing the solver a triangle mesh of its own would duplicate several
hundred megabytes, would need a second acceleration structure, and would lose the per-
triangle *material* that the whole game depends on — footstep sounds, bullet decals, damage
falloff and AI audibility are all keyed to the material of the triangle that was hit. Keeping
the level in the engine's own database and answering the solver from it keeps one copy, one
tree, and the material on every contact.

Everything difficult about colliding with a triangle soup lives here: caching, tunnelling,
degeneracy at shared edges, and deciding what a body that is *already inside* the geometry
should be pushed toward.

## Where it sits

It rests on the static collision database ([`src/xrCDB`](../../xrCDB/README.md)), on the
material library ([`src/xrMaterialSystem`](../../xrMaterialSystem/README.md)) for the flags
carried by each triangle, on the shape payload in
[`../ExtendedGeom.h`](../ExtendedGeom.h.md) where every shape's cache and history are kept,
and on [`../dcylinder`](../dcylinder/README.md) for the cylinder primitive it must collide.
Nothing in the engine depends on it directly: it is reached only through the solver's
narrow-phase dispatch, which is why a rebuild can replace it wholesale with its library's own
mesh collider — at the cost of solving every problem listed below again.

The directory's name preserves the provenance: this began as a third-party triangle-list
collider for the dynamics library and was rewritten around the engine's own database. The
file `dTriCallideK.cpp` is misspelled in the original and the recipe keeps the spelling so
the two trees correspond.

## The load-bearing ideas

Read these once here; the twins assume them.

**A shape kind is a small vtable.** The collider is registered with the dynamics library as a
new kind of shape: a byte count for its private data, a bounds function (which reports
*infinite* bounds, because the level is everywhere), a function that returns the narrow-phase
routine for a given other kind, and a destructor. Only three other kinds are answered — box,
sphere and cylinder — and everything else silently does not collide with the level. Capsules
and planes are absent because no shape in the shipped game is one.

**Per-shape triangle cache.** Querying the database costs a tree descent, and a body at rest
would pay it sixty times a second for the same answer. So every shape remembers the triangles
returned by its last query and the enlarged box that query covered; while the shape's current
box stays inside the cached box, the cached triangle list is reused. The enlargement factor is
a console variable, which makes the trade — query cost against cache staleness — tunable at
run time. The query box is also grown by the shape's velocity times a fixed lookahead, so a
moving body's cache covers where it is going.

**Positive and negative triangles.** Every candidate triangle is reduced to a plane, and the
shape is either in front of that plane (*positive*) or behind it (*negative*). The two are
handled by completely different code paths, and this is the distinction that organizes the
whole algorithm. A positive triangle is an ordinary obstacle: run the primitive-versus-
triangle routine and emit whatever contacts it finds. A negative triangle means the shape's
centre is already behind the surface — it has penetrated, or it is standing on the back of a
one-sided wall — and the question is no longer "where do they touch" but "which way is out".

**Only one negative triangle wins.** Among all negative triangles the collider keeps the one
of least penetration depth and generates a single contact against its plane. Emitting a
contact per negative triangle would give a body inside a corner several mutually contradictory
escape directions and eject it at speed. Passable materials (foliage, cloth) are tracked in a
second, parallel slot, so a body that is inside a bush *and* inside a wall gets one contact for
each rather than losing one to the other.

**Persistence across steps: the pushing state.** Which negative triangle was chosen last step
is remembered per shape, along with the shape's previous position. That memory does three
things. It gives hysteresis, so a body being pushed out of a wall keeps being pushed by the
same face rather than oscillating between two. It provides the *opposite-side veto*: a
candidate whose normal opposes the currently pushing normal by more than 135 degrees is
rejected outright, which is what stops a shape inside thin geometry from being pushed
alternately out of each side. And it supplies the previous position needed for the next idea.

**Tunnelling is caught by the segment, not by the shape.** For each negative triangle, the
segment from the shape's previous position to its current position is intersected with the
triangle's plane, and the crossing point is tested for containment in the triangle. If the
shape crossed the triangle between steps, it passed *through* the surface, and the contact
generated is not a plane contact at all: it is a contact whose normal points back along the
shape's own trajectory, with a depth equal to the distance travelled. The body is pushed back
the way it came. This is cheaper and more robust than sweeping the shape, and it is the
mechanism behind conformance criterion 9's "no tunnelling" — see also
[`../dRayMotions.h`](../dRayMotions.h.md) and
[`../PHMoveStorage.h`](../PHMoveStorage.h.md), which supply the swept-motion path for
the cases this one cannot cover.

**Degeneracy at shared edges.** A triangle soup has no notion of a surface, so a body sliding
across the join between two coplanar triangles meets the *edge* of the second one and is
kicked. The rule used here is a neighbour test: a negative triangle is discarded when a
positive triangle that contains the shape's centre either shares all three of its vertices, or
holds a vertex the negative triangle's plane says should be behind it. Read plainly: if a
face the shape is legitimately in front of contradicts the negative face's verdict, the
negative face is an interior edge and is not a real surface. This is the internal-edge, or
"ghost collision", problem, and every triangle-soup collider must solve it somehow; a rebuild
using a library mesh collider must check that its library does.

**Contacts carry materials, not parameters.** The narrow phase writes the triangle's material
index into the contact and stops. Friction, bounce and softness are derived later from the
*pair* of materials involved — see [`../Physics.cpp`](../Physics.cpp.md) and
[`src/xrMaterialSystem`](../../xrMaterialSystem/README.md). Keeping the two apart is what
lets one collision routine serve metal on concrete and flesh on mud.

**The contact budget is a hard limit with a margin.** The caller states how many contacts it
will accept; the loop stops ten short of it and reserves the remainder for the negative-
triangle contacts, which are emitted last and matter most. A body that runs out of budget
loses positive-triangle contacts — places it is merely touching — and keeps the ones that are
pushing it out of trouble.

**Determinism.** The triangle order comes from the database query and the reductions are
order-dependent (least depth wins, first match vetoes). Conformance criterion 8 therefore
requires the database to return triangles in a stable order for a stable query. A rebuild that
parallelizes the query, or that uses an unordered container anywhere in this directory, breaks
physics determinism without breaking anything visible until two machines disagree.

## Twins

| File | Role |
|---|---|
| [`dTriList.h`](dTriList.h.md) | The public face: the shape kind's identifier, the per-triangle and per-object callbacks the engine installs, and the constructor. |
| [`dTriList.cpp`](dTriList.cpp.md) | Registers the shape kind with the dynamics library, dispatches by the other shape's kind, and applies the per-shape veto on colliding with the level at all. |
| [`dxTriList.h`](dxTriList.h.md) | The shape kind's private data — the two callbacks and the collider instance — plus a local three-float vector type it carries for historical reasons. |
| [`dcTriListCollider.h`](dcTriListCollider.h.md) | The collider itself: three public entry points (box, sphere, cylinder) over a large private surface of per-primitive projections, containment tests and contact builders. |
| [`dcTriListCollider.cpp`](dcTriListCollider.cpp.md) | Computes each primitive's world-aligned extent, grows it by the body's velocity, and enters the shared sweep. |
| [`dSortTriPrimitive.h`](dSortTriPrimitive.h.md) | The heart of the directory: the cache, the positive/negative classification, the pushing state, the tunnelling test, the degeneracy rule and the contact emission — one routine, parameterised by primitive. |
| [`dSortTriPrimitive.cpp`](dSortTriPrimitive.cpp.md) | Nothing. The translation unit exists so the build system has a file; its content is disabled. |
| [`TriPrimitiveCollideClassDef.h`](TriPrimitiveCollideClassDef.h.md) | Mints, for each primitive, the tiny adaptor the shared sweep is parameterised over: project onto an axis, collide with a triangle, collide with a triangle's plane. |
| [`dcTriangle.h`](dcTriangle.h.md) | The working record a candidate triangle is reduced to: two side vectors, a normal, the shape's signed distance from the plane, the plane's offset, a penetration depth and the database triangle it came from. |
| [`dTriColliderMath.h`](dTriColliderMath.h.md) | Point-in-triangle by three edge half-space tests, the plane-crossing point of a segment, and the triangle reduction itself. |
| [`dTriColliderCommon.h`](dTriColliderCommon.h.md) | The striding arithmetic that lets the routines write into the caller's interleaved contact array, the contact-count mask, and two trigonometric constants. |
| [`__aabb_tri.h`](__aabb_tri.h.md) | The box-versus-triangle overlap test used to reject candidates before any real work, in two strengths: a cheap approximate one and the full separating-axis one. |
| [`dTriBox.h`](dTriBox.h.md) | A box's extent along an arbitrary axis, and the line-and-box helpers the box-versus-triangle manifold is built from. |
| [`dTriBox.cpp`](dTriBox.cpp.md) | Box against a triangle, and box against a triangle's plane — the largest of the three narrow phases, because a box meets a triangle at faces, edges and corners. |
| [`dTriSphere.h`](dTriSphere.h.md) | A sphere's extent along an axis: its radius, whatever the axis. |
| [`dTriSphere.cpp`](dTriSphere.cpp.md) | Sphere against a triangle and against a triangle's plane — the simple case, and the one the others are checked against. |
| [`dTriCylinder.h`](dTriCylinder.h.md) | A cylinder's extent along an arbitrary axis, from the angle between the axis and the cylinder's own. |
| [`dTriCylinder.cpp`](dTriCylinder.cpp.md) | Cylinder against a triangle and against a triangle's plane — the case every character in the game depends on, including the rim contacts that let a character stand on an edge. |
| [`dTriCollideK.h`](dTriCollideK.h.md) | An umbrella that pulls in the three narrow phases together. |
| [`dTriCallideK.cpp`](dTriCallideK.cpp.md) | The translation unit that materializes that umbrella. The filename's misspelling is the original's. |
