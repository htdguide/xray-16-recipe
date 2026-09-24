# src/xrPhysics/PHGeometryOwner.cpp

> Owns a set of collision shapes as one composite: builds them into a shared group, propagates material and callbacks across them, and derives volume, mass centre and extent from the set.

**Needs** — [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`Geometry.h`](Geometry.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [`xrMaterialSystem`](../xrMaterialSystem/README.md)
**Used by** — [`PHGeometryOwner.h`](PHGeometryOwner.h.md)
**Tier floor** — T2: shape assembly and mass arithmetic; the shapes themselves are the layout-bound part and live in [`Geometry.cpp`](Geometry.cpp.md).

## Purpose

One rigid body in this engine is rarely one shape. A bone carries a box, a sphere or a cylinder;
a breakable object's root body carries the shapes of every rigid child bone welded into it; a door
carries one box. This file is the answer to "several shapes, one body": how they are grouped so the
collider sees them as one, how a property set on the composite reaches every member, and how the
composite's mass properties follow from its members' volumes.

The load-bearing idea is the **group**: an inner collision space created lazily on first use and
destroyed when it empties. A group is itself a shape from the collider's point of view, so a
composite of twelve shapes is presented to the broadphase as a single entry, and the collider only
descends into it once the outer test passes.

## State

```text
RECORD GeometryOwner
  shapes            : list<CollisionShape>     # owned; destroyed with this record
  group             : optional<collision group shape>   # created on first add, destroyed when empty
  built             : bool                     # shapes exist in the world, not merely described
  mass_center       : vector                   # in the owner's local frame
  volume            : real                     # sum of the members' volumes
  material          : material id              # applied to every member
  contact_callback  : optional<StaticContactHook>
  object_callback   : optional<ObjectContactHook>
  ref_object        : optional<owning game entity>

  # invariant: `built` false means the shapes are described but not registered anywhere;
  #            most setters silently record and defer until build
  # invariant: group exists if and only if at least one shape has been added to it
  # invariant: each shape records its own index within `shapes`; the index is kept correct
  #            when shapes are moved between owners (see PHElement's geometry transfer)
```

## `build` / `destroy`

**Contract** — `build` finalises every described shape against the composite's mass centre, stamps
it with the shared material, callbacks and owning entity, records its index, and adds it to the
group. Idempotent. `destroy` tears the shapes down but keeps their descriptions, so the same owner
can be rebuilt.

```text
FUNCTION build()
  IF built THEN RETURN
  FOR index, shape IN shapes
    shape.realise(relative_to := mass_center)
    shape.material := material
    IF contact_callback EXISTS THEN shape.contact_callback := contact_callback
    IF object_callback  EXISTS THEN shape.object_callback  := object_callback
    IF ref_object       EXISTS THEN shape.ref_object       := ref_object
    group_add(shape)
    shape.index_in_owner := index
  built := true
```

**Invariants** — every shape is realised *relative to the mass centre*, not to the bone origin.
That is what makes the body's origin coincide with its centre of mass, which the dynamics library
requires and which is why `mass_center` must be final before `build` runs.

## `add_box` / `add_sphere` / `add_cylinder` / `add_shape`

**Contract** — describe a new shape. `add_shape` dispatches on the authored bone-shape kind and
optionally applies a frame offset first — the offset path is how a rigid child bone's shape is
folded into its parent's body. Adding does not build.

**Notes** — box half-sizes are **clamped to a 5 mm floor** on every axis. Authored models contain
degenerate bone shapes (zero thickness on some axis), and a zero-extent box makes the collider
produce garbage normals. Five millimetres is below the smallest thing the player can notice and
above the collider's tolerance. Keep the clamp; the shipped data depends on it.

## `get_mc_data` — mass centre from volume

**Contract** — sets the composite's mass centre to the volume-weighted average of its shapes'
centres, and records the total volume. Assumes uniform density.

```text
FUNCTION mass_centre_from_volumes() -> vector
  total := 0 ; weighted := 0
  FOR EACH shape IN shapes
    v := shape.volume()
    weighted := weighted + shape.local_center() · v
    total    := total + v
  volume      := total
  mass_center := weighted / total
```

## `get_mc_kinematics` — mass centre from the model

**Contract** — the alternative: take the mass and mass centre *authored per bone* in the model file
and combine them, ignoring the shapes' geometry except to accumulate volume.

```text
FUNCTION mass_centre_from_model(skeleton) -> (centre, mass)
  mass := 0 ; centre := 0 ; volume := 0
  FOR EACH shape IN shapes
    bone := skeleton.bone_data(shape.bone_id)
    mass   := mass + bone.mass
    volume := volume + shape.volume()
    centre := centre + bone.center_of_mass · bone.mass
  centre := centre / mass
```

**Notes** — the two paths coexist because both kinds of data ship. A model with authored per-bone
mass uses the second; a shell built from bare shapes (a door, a box dropped by a script) uses the
first with a density. Which is used is decided by whether a skeleton is present at the moment mass
is reset; see `re_adjust_mass_positions` in [`PHElement.cpp`](PHElement.cpp.md). A rebuild that
merges them will get either the authored ragdoll tuning wrong or the scripted-object masses wrong.

## `set_material` / `set_contact_callback` / `set_object_contact_callback` / `set_ref_object`

**Contract** — record the property on the composite and, if already built, push it to every member.
Each is a one-line rule and they all share it: *set-before-build is remembered, set-after-build is
propagated*. `add_object_contact_callback` and `remove_object_contact_callback` are the multi-hook
variants — a shape may carry several object callbacks at once, while the composite only remembers
one as its "primary". That asymmetry is a wart: adding a second callback and then removing the
first leaves the composite's record pointing at a callback that is no longer primary anywhere.

## `spaced_geometry`

**Contract** — the single shape handle representing the whole composite to the outside world: the
group. Returns nothing before the composite is built.

## `get_extensions` / `get_max_area_dir` / `get_radius`

**Contract** — extent along an axis (union over members), the direction of the largest projected
face of the *first* member, and the radius of the *last* member. The latter two are unapologetic
approximations used where a rough answer is enough (deciding which way a crate wants to lie, sizing
a debug marker).

## `set_static_form` / `set_position` / `clear_cashed_tries` / `clear_motion_history`

**Contract** — place the composite as an immovable object at a frame; reposition its shapes about a
new local origin; and the two forget-what-you-knew operations. The cached triangle sets and motion
history exist per shape because the triangle-mesh collider caches which triangles it last touched
(a large win for a resting object) and remembers the previous placement for swept tests. Both
caches must be dropped whenever the object is teleported, or the next step will sweep across the
whole level.

## `add_geom` / `remove_geom` / `geom_by_bone_id`

**Contract** — add or remove a single shape from a *built* composite, keeping the group's
membership correct and destroying the group when it empties; and find a member by the skeleton bone
it came from. Used by breakables, which move shapes between bodies at runtime.
