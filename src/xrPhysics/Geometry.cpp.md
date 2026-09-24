# src/xrPhysics/Geometry.cpp

> Builds, places, masses and queries the primitive collision shapes that make up
> every physical object in the game.

**Needs** — [`Geometry.h`](Geometry.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`dcylinder/dCylinder.h`](dcylinder/dCylinder.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Geometry.h`](Geometry.h.md)
**Tier floor** — T1: shapes are allocated by the dynamics library, placed by writing its
matrices directly, and their mass tensors are accumulated in its representation.

## Purpose

A physical object is a set of bodies, and a body is a set of shapes. This file is the shape
layer: it takes the box, sphere and cylinder descriptions authored in a model's skeleton
data and turns each into a live collision shape that follows a body, carries a material and
a bone id, and can report its mass contribution and its extent along any axis.

Two things here are load-bearing beyond "wrap a library call": the **two-level shape
representation** and the **mass accumulation into a common reference point**.

## State

```text
RECORD Geom                           # abstract; each shape kind adds its own description
  transform_shape : Shape             # the outer shape: holds the LOCAL placement
  inner_shape     : Shape             # the actual box / sphere / cylinder
  bone_id         : int (16-bit)      # skeleton bone this shape follows; none = unset
  flags           : bit set           # shape flags copied from the skeleton data
  # invariant: the payload record (ExtendedGeom) lives on inner_shape, never on the wrapper
  # invariant: while a shape exists, exactly one of the two may be attached to a body
```

## the two-level representation

**Contract** — every shape is created as an inner primitive wrapped in a transform shape.
The body is attached to the *wrapper*; the primitive carries the offset and rotation that
places it in the body's frame.

**Notes** — the reason is that the underlying library attaches a shape to a body at the
body's origin and nowhere else, while a skeleton describes a collision shape at an arbitrary
offset from the bone. The wrapper supplies the missing degree of freedom.

The cost is paid on every access: the shape's true world transform is the wrapper's
transform composed with the primitive's, and *both* the payload record and the material live
on the primitive, not the wrapper. Every accessor in this file therefore branches on whether
the wrapper has an inner shape at all — a run of near-identical branches that is entirely an
artifact of this representation. A rebuild whose shapes carry their own local offset deletes
the wrapper, the branch, and the whole class of bugs where code reads the wrapper's payload
and finds nothing.

```text
FUNCTION world_transform(geom) -> matrix
  IF geom.transform_shape is a wrapper
    RETURN compose( wrapper.rotation, wrapper.position,
                    inner.rotation,   inner.position )
  RETURN transform of the single shape
```

## mass accumulation

**Contract** — a shape reports its mass tensor about a caller-supplied reference point, at
unit density or at a given density, and can add itself into a running total.

```text
FUNCTION add_self_mass(total, reference_point, density)
  m = unit_density_mass_of_this_shape()       # per shape kind, closed form
  scale m so its total equals density * volume_of_this_shape()
  translate m from this shape's own centre to reference_point
  total = total + m                           # tensors add about a COMMON point
```

**Invariants** — every shape contributing to one body must be translated to the *same*
reference point before adding, or the resulting inertia tensor is meaningless. The reference
point used during construction is the body's intended centre of mass.

**Notes** — the per-shape unit-density tensors are closed forms (box from its side lengths,
sphere from its radius, cylinder from radius and height about its own axis), and the box and
cylinder additionally rotate their tensor into the shape's local orientation before
translating. Getting that rotation step wrong produces a body that tumbles when it should
not, which is a symptom worth knowing about because nothing asserts it.

Volume is computed analytically per shape and is what turns an authored *density* into a
mass. The engine's own densities come from the material library, so the same visual model
with a different material assignment weighs differently.

## extent along an axis

**Contract** — project the shape onto a given world axis and return the low and high bounds
relative to a supplied origin along that axis. Implemented per shape kind:

```text
box:       half_extent = sum over the box's three local axes of
                         |dot(axis, local_axis)| * side_length / 2
sphere:    half_extent = radius
cylinder:  half_extent = |dot(axis, cylinder_axis)| * height / 2
                       + sin_of_that_angle * radius
```

**Notes** — this is the primitive out of which an object's bounding box is built for any
orientation, and it is why the wrapper exposes it as a virtual: the box that a shell reports
is the box that just contains every one of its shapes along three chosen axes, not an
axis-aligned box of its shapes' axis-aligned boxes. The difference matters for the
activation procedures, which push an object out of walls using exactly this box.

## build and destroy

**Contract** — building creates the primitive, wraps it, attaches a fresh payload record,
stamps the bone id, and places the primitive at its authored offset *relative to the
supplied reference point* — normally the body's centre of mass, so that the body's origin
and its centre of mass coincide. Destroying releases the payload records of both levels and
the shapes themselves, and leaves the wrapper cleared.

**Invariants** — the wrapper is created with automatic cleanup *disabled*, so destroying the
wrapper never destroys the primitive; this file owns both and releases them in order. In a
rebuild this is just: one owner, two resources.

## motion history

**Contract** — clearing a shape's motion history either marks it unspecified (the next step
treats the shape as having teleported) or seeds it with the shape's current world centre
(the next step treats the shape as continuous). Called after every explicit placement.

**Notes** — the choice between the two is the whole reason this entry point takes a flag. A
shape whose owner teleported must *not* be swept from its old position, because the sweep
would collide with everything along the way; a shape that was merely rebuilt in place must
be, or it will tunnel on its first step. Getting this wrong produces either objects that
snag on teleport or objects that fall through floors on spawn.

## callbacks, material and ownership

**Contract** — setting a shape's material, contact callbacks, callback data, owning game
object or owning physics object all write into the payload record, on the primitive when
there is one and on the wrapper otherwise. Adding and removing object-contact callbacks
appends to and removes from the chain rather than replacing it.

**Notes** — the *setter* for object-contact callbacks replaces the whole chain while the
*adder* appends. Both exist and the difference is meaningful: a shell setting its callback
means "this is the shape's behaviour", while a subsystem adding one means "and also watch
this shape". Confusing them silently drops another subsystem's hook.

## fluid collision

**Contract** — a shape reports whether it participates in fluid (fog volume) collision, which
is a single flag from the authored skeleton data, inverted. The default is to participate.

## debug rendering

**Contract** — debug-only: each shape kind draws itself as a wireframe at its true world
transform, plus its centre and its axes. Purely observational.
