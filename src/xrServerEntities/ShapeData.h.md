# src/xrServerEntities/ShapeData.h

> The volume an entity occupies when it is not a model: a list of spheres and boxes, as stored in every restrictor, zone and smart terrain record.

**Needs** — [`xrCore/_sphere.h`](../xrCore/_sphere.h.md) · [`xrCore/_matrix.h`](../xrCore/_matrix.h.md)
**Used by** — [`object_factory_spawner.cpp`](object_factory_spawner.cpp.md) · [`xrServer_Objects.cpp`](xrServer_Objects.cpp.md) · [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) · [`xrServer_Objects_Alife_Smartcovers.cpp`](xrServer_Objects_Alife_Smartcovers.cpp.md)
**Tier floor** — T1: the shape list is serialized into spawn records and saves at a fixed layout.

## Purpose

A large class of entities in this world *is* a volume and nothing else — a restrictor, an
anomaly, a smart terrain's catchment, a team base, a script-defined trigger region. Their
extent is authored in the level editor as a handful of primitives, and this is the record
that carries them. It is in its own file because both the record classes and the level tools
need it and neither should pull in the other.

## State

```text
ENUM ShapeKind
  sphere = 0
  box    = 1

RECORD Shape
  kind : int (8-bit)      # the ShapeKind tag; the width is what the stream writes
  body : sphere OR box    # exactly one, selected by kind

RECORD ShapeSet
  shapes : list<Shape>
```

A **sphere** is a centre and a radius. A **box** is a full affine transform — the unit cube
mapped by that transform — so it carries orientation and non-uniform scale, not just extents.

**Invariants**

- The kind tag is one byte and its two values are frozen: they appear in every shipped
  spawn record that has a shape.
- The sphere and the box share storage, and the tag is the only thing that says which is
  live. Reading a box's bytes as a sphere is silent nonsense, so the tag must be read first
  and trusted.
- An entity's effective volume is the **union** of its shapes. The set is unordered; nothing
  depends on the sequence.
- An empty set is legal and means the entity has no extent. Something creating a zone
  programmatically must supply a shape or the zone affects nothing — see the default sphere
  in [`object_factory_spawner.cpp`](object_factory_spawner.cpp.md).

## Notes

**Why a box is a matrix and not two corners.** Level authors rotate their trigger volumes.
Storing the transform means the containment test is "map the point through the inverse and
test against the unit cube", which is one matrix apply and three comparisons — and it means
the editor's gizmo and the runtime test agree exactly, because they share the transform
rather than deriving bounds from it.

**Why this is not the collision database.** These volumes are *logical* — "am I inside the
anomaly", "may this creature path here" — and are tested against entity positions a few
times a second, not against geometry millions of times a frame. Chapter 7's tree is for the
world; this is for regions. Keeping them separate is why a level can have hundreds of
restrictors at no rendering cost.
