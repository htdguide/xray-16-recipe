# src/xrGame/IKFoot.cpp

> One foot's geometry and its ground contact: where the toe and heel are, which way the sole faces, and what rotation and shift of the leg's last bone would put the foot flat on the surface under it.

**Needs** — [`IKFoot.h`](IKFoot.h.md) · [`IKFoot_inl.h`](IKFoot_inl.h.md) · [`ik_calculate_data.h`](ik_calculate_data.h.md) · [`ik_foot_collider.h`](ik_foot_collider.h.md) · [`ik_collide_data.h`](ik_collide_data.h.md) · [`GameObject.h`](GameObject.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [`xrEngine/EnnumerateVertices.h`](../xrEngine/EnnumerateVertices.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`IKFoot.h`](IKFoot.h.md); callers name that, not this file.
**Tier floor** — T2: vector and plane geometry over a mesh's bind pose; nothing device-facing

## Purpose

Inverse kinematics for feet needs to know what a foot *is*, and the model does not say.
This file derives it: given the four bones of a leg, it finds the toe tip and the heel by
**walking the skinned vertices of the foot and toe bones** and taking the extreme vertex
along a derived axis. Everything afterwards — the ground query, the rotation that lays the
sole flat, the shift that plants it — is expressed against those two points.

Deriving the geometry from the mesh rather than authoring it is the load-bearing decision.
It means foot placement works on every creature in the game without per-model tuning, and
it means a rebuild must be able to enumerate a bone's influenced vertices in bind pose.

The second decision worth naming is the **reference bone**. A foot is two bones: the ankle
and the toe. Which of them the placement is expressed relative to depends on the model —
some skeletons rotate the toe under the ankle and some do not — and the answer is either
configured or detected by comparing the two bones' sole normals. Every conversion in this
file exists to move a transform between "expressed at the ankle" and "expressed at the
toe".

## State

```text
RECORD LocalVector                 # a vector, and which bone's space it is in
  v    : vec3
  bone : int          # 2 = ankle, 3 = toe; the leg's four bones are indexed 0..3

RECORD IKFoot
  skeleton        : Kinematics
  ref_bone        : int          # 2 or 3; which bone the goal is expressed at
  foot_bone_id    : int          # the ankle's actual bone identifier in the skeleton
  toe_bone_id     : int
  toe_position    : LocalVector  # derived: the extreme vertex toward toe-and-sole
  heel_position   : LocalVector  # derived: the extreme vertex toward heel-and-sole
  foot_normal     : LocalVector  # which way the sole faces
  foot_direction  : LocalVector  # which way the toes point
  bind_ankle_to_toe : matrix     # the bind-pose relation between the two bones
  foot_width      : real         # sole-normal distance from the ankle to the toe point
```

Invariants:

- Every stored vector records the bone it is expressed in, and conversions between the
  two are done through the *bind pose* relation, not the current pose. A vector in the
  ankle's space and the same vector in the toe's space differ by a fixed transform.
- The toe and heel points are in different spaces by construction: the toe ends up in the
  reference bone's space, the heel stays in the ankle's. Every consumer must use the right
  conversion, which is why the accessors exist at all.

## `Create`

**Contract** — bind the foot to a skeleton and a configuration section, choosing the
reference bone and the sole normal and toe direction. Defaults exist for both reference
bones; a section overrides all three. Then derives the toe and heel.

**Invariants** — the configuration key that selects the toe as the reference bone is
literally "align to the toe", which is what it means: the placement will lay the *toe*
flat rather than the ankle. The sole normal and toe direction are then read in *that*
bone's space, which is why the reference bone must be decided first.

## `set_toe` — deriving the foot's geometry

**Contract** — find the toe tip and the heel from the mesh. Runs once, at creation,
against the bind pose.

**Invariants** — three separate extremes are taken and combined, and the combination is
the whole algorithm:

- the **toe tip** is the vertex of the *toe bone* furthest along the sum of the sole
  normal and the toe direction — that is, furthest toward the front-and-bottom corner;
- the same search is then run over the *ankle bone* along the sole normal alone, and the
  toe point takes the greater of the two along one axis, so that a model whose toe bone is
  short still gets a toe point at the front of the foot;
- the **heel** is the vertex of the ankle bone furthest along the sole normal *minus* the
  toe direction — the back-and-bottom corner — and is then nudged **forward by a fifth of
  the foot's length**.

```text
FUNCTION derive_foot_geometry(leg_bones)
  binds = the skeleton's bind transforms
  bind_ankle_to_toe = inverse(bind of ankle) composed with bind of toe
  normal, direction = the sole normal and toe direction, both in the ankle's space

  axis = normalize(normal + direction)              # front-and-bottom
  toe  = the toe bone's vertex furthest along axis, in the toe bone's bind space
  toe  = toe expressed in the ankle's space

  axis = normal                                     # straight down the sole
  p    = the ankle bone's vertex furthest along axis
  toe  = toe, with its sole-axis component raised to p's if p reaches further

  axis = normalize(normal - direction)              # back-and-bottom
  heel = the ankle bone's vertex furthest along axis
  heel = heel + direction * (foot length along direction) * 0.2

  toe  = toe expressed in the reference bone's space
  foot_width = the sole-normal distance from the ankle's origin to the toe point
```

**Notes** — the heel's forward nudge of one fifth is the only tuned number here and its
purpose is not stated. It moves the heel contact point inside the sole rather than at its
very back edge, which stops the foot from pivoting on a corner when the character stands
on a slope. A rebuild should keep it and may want it configurable.

The "raise the toe's sole component to the ankle's extreme" step means the toe point is
not necessarily a real vertex of either bone; it is a corner of the foot's silhouette.
That is deliberate — the contact point wanted is the front of the *sole*, not the tip of
the mesh.

## `SetFootGeom`

**Contract** — build the three world points the ground query needs: the toe, the heel, and
a **side** point. The side point is placed at the midpoint of toe and heel, offset
perpendicular to both the sole normal and the toe direction by the foot's own length.

**Invariants** — the offset is scaled to the toe-to-heel distance, so the query's triangle
is roughly as wide as the foot is long, regardless of the creature's size. A dog's foot
and a human's produce proportionally similar query shapes without any per-creature number.

## `Collide`

**Contract** — build the foot's query geometry in world space and hand it to the collider,
which fills in a contact record: whether anything was hit, the plane of what was hit,
which of the three points made contact, and the direction the query was cast in. The
foot-step flag distinguishes a planting foot from one merely being placed.

## `GetFootStepMatrix` — the placement

**Contract** — given the animation's transform for the reference bone and a contact
record, produce the transform that puts the foot on the surface, together with a
classification of *how* it had to move. Reports whether any correction was made at all.

**Invariants** — the correction is refused when nothing was hit, or when the contact point
is further from the surface than the maximum correction distance. A foot in mid-air is
left exactly where the animation put it; foot placement never invents motion, it only
corrects small errors.

The **contact point depends on which part of the foot touched**: a toe contact is
corrected about the toe, a heel or side contact about the heel. That is why the heel is
derived at all — a character stepping backwards down a slope pivots about the heel and
looks wrong if corrected about the toe.

The result carries one of five states — free, aligned, rotational, translational or mixed
— and the state is what the controller above uses to decide how much to blend the
correction in. A purely translational correction can be applied fully; a rotational one
must be eased.

```text
FUNCTION foot_step_matrix(animation_transform, contact, may_collide, may_rotate)
  contact_point = the toe, in the reference bone's space
  IF the contact was on the heel or the side THEN
    contact_point = the heel, converted into the reference bone's space
  world_point  = animation_transform applied to contact_point
  world_normal = animation_transform applied to the sole normal

  distance = signed distance from world_point to the contact plane
  IF nothing was hit OR |distance| > the maximum correction THEN
    result = the animation transform, state free ; RETURN "no correction"

  transform = animation_transform
  IF may_rotate THEN state = rotate_to_plane(transform, plane, world_normal,
                                             world_point, may_collide)
  IF shift_to_plane(transform, contact_point, may_collide, plane, cast direction) THEN
    promote the state: free or undefined -> translational; rotational -> mixed;
                       aligned stays aligned
  ELSE IF the state is still undefined THEN state = free
  result = (transform, state) ; RETURN "corrected"
```

## `rotate` — laying the sole flat

**Contract** — rotate the foot about the axis perpendicular to both the sole normal and
the surface normal, by the angle between them, **clamped to thirty degrees**, keeping the
origin fixed. Optionally passes the angle through the collision check first.

**Invariants** — the clamp is what stops a foot from rotating to match a wall. Beyond
thirty degrees the surface is not something the character can stand flat on, and the
animation's own orientation is better than a correct-but-absurd one.

The rotation is applied in *foot* space and converted back to the reference bone, which
is where the ankle/toe conversions earn their keep: rotating the reference bone directly
would rotate about the wrong pivot when the toe is the reference.

## `CollideFoot` — how far it may rotate before the foot goes through the ground

**Contract** — given a proposed rotation, decide whether the foot is already flat enough,
whether the rotation is free, or whether it must be reduced — and by how much.

**Invariants** — "already flat enough" is decided by comparing the ankle's height above
the surface against the foot's own half-width projected onto the surface normal. A foot
whose ankle is closer to the ground than the sole is thick is already in contact and must
not be rotated further.

The reduced angle is the difference between two arc-cosines: the angle the toe currently
subtends about the rotation axis, and the angle it would subtend at the surface. That is
the exact rotation that brings the toe onto the plane rather than through it.

## `make_shift` — planting the foot

**Contract** — translate the foot along the ground query's cast direction until the
contact point lies on the surface. Refuses the shift outright when the foot would have to
move *up* and collision is being respected — a foot may sink to meet the ground but may
not be lifted through it.

**Invariants** — when the cast direction is nearly parallel to the surface the shift
distance diverges, so the direction is bent toward the surface normal until their dot
product reaches a minimum. The shift is then clamped to the maximum correction distance in
both directions.

**Notes** — the minimum dot product is nine tenths, with a comment recording that it was
once the cosine of forty-five degrees. Tightening it makes the shift direction closer to
the surface normal and therefore shorter and better behaved; loosening it preserves more
of the original cast direction. Neither value is derived.

## `get_ref_bone` / `set_ref_bone`

**Contract** — detect which bone the placement should be expressed at, by transforming the
sole normal through both the ankle's and the toe's *current* transforms and comparing. If
they disagree by more than a small angle, the toe is bent relative to the ankle and the
toe must be the reference; otherwise the ankle will do.

**Invariants** — this is evaluated against the live pose, not the bind pose, so a
character whose toe bends during an animation switches reference bone mid-motion. That is
intended: whether the foot is one rigid plate or two depends on what the animation is
doing.

## The conversions — `ref_bone_to_foot`, `foot_to_ref_bone`, and their transform forms

**Contract** — move a transform between the reference bone's frame and the ankle's. When
the ankle *is* the reference bone, all four are the identity and return their argument
unchanged; otherwise the relation is taken from the two bones' current transforms.

**Invariants** — the relation uses the **current** pose, not the bind pose, unlike the
vector conversions which use the bind pose. The distinction matters: a vector attached to
the foot is fixed in the foot, while a frame conversion must follow the animation.

## The accessors — `ToePosition`, `HeelPosition`, `FootNormal`, `get_local_vector`

**Contract** — hand out each stored vector *in the reference bone's space*, converting
through the bind relation when the vector is stored in the other bone's space. Anything
else is a programming error and is asserted.

**Notes** — the heel accessor is the exception: it returns its stored value without
conversion, with the conversion path commented out. Every caller that needs the heel in
another space converts explicitly. This asymmetry between the toe and the heel accessors
is easy to miss and is exactly the sort of thing a rebuild should normalize.
