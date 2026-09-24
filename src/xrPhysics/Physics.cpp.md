# src/xrPhysics/Physics.cpp

> Contact generation: turn a colliding pair into constraints whose friction, bounce and softness come from the two surfaces' material records, letting both sides veto or reshape each contact before it is committed.

**Needs** — [`Physics.h`](Physics.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`tri-colliderknoopc/dTriList.h`](tri-colliderknoopc/dTriList.h.md) · [`xrMaterialSystem`](../xrMaterialSystem/README.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`Physics.h`](Physics.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md)
**Tier floor** — T1: writes contact records the dynamics library reads back by layout, and reaches into body and constraint internals for the inertia tensor.

## Purpose

This is the seam the preface describes when it says the stepping model must *separate collision
detection from the constraint solve, because the engine injects its own contacts between the two*.
Everything the game means by "what a surface feels like" — mud that drags, water that slows,
foliage you walk through, a floor that bounces a grenade — is decided in one loop in this file.

The shape of the decision: the collider hands back raw contact points; the engine then, for each
point, looks up both surfaces' material records, derives the constraint parameters from the
*pair*, lets any number of registered callbacks amend or veto the contact, and only then commits it.

## State

Module-level, shared per step:

```text
ContactGroup        : pool of contact constraints, emptied at the end of every step
ContactFeedBacks    : pool of force-feedback records, emptied with it
ContactEffectors    : pool of per-body accumulated contact effects, emptied with it
world_boundaries    : box; anything below its floor is considered escaped
```

Plus the global tuning constants documented in [`PhysicsCommon.h`](PhysicsCommon.h.md), which are
*defined* here.

## `collide_into_group` — the contact loop

**Contract** — collide two shapes, and commit up to a caller-supplied budget of contacts into the
given island. Returns how many were committed. Uses one fixed-size scratch buffer, so it allocates
nothing per call; a pair producing more contact points than the buffer holds is silently truncated.

```text
FUNCTION collide_into_group(shape_a, shape_b, island, max_contacts) -> int
  points := collider.contacts(shape_a, shape_b, buffer_capacity)   # raw geometry only
  committed := 0

  FOR EACH contact IN points
    # --- 1. establish what the two surfaces are -----------------------------
    data_a := user_data_of(contact.shape_a)      # none for a raw triangle
    data_b := user_data_of(contact.shape_b)
    material_a := data_a.material IF data_a EXISTS ELSE default
    material_b := data_b.material IF data_b EXISTS ELSE default
    # a triangle-soup shape reports its per-triangle material through the contact's
    # mode field, which the collider overloads for exactly this purpose
    IF shape_a IS triangle_soup THEN material_a := contact.mode_field
    IF shape_b IS triangle_soup THEN material_b := contact.mode_field
    IF neither side is triangle_soup THEN clear contact.mode_field

    # --- 2. derive constraint parameters from the material PAIR -------------
    contact.mode := approximate_friction_pyramid | soft_error_reduction | soft_force_mixing
    spring  := material_a.spring  · material_b.spring  · world_spring
    damping := material_a.damping · material_b.damping · world_damping
    contact.soft_erp := erp(spring, damping, fixed_step)
    contact.soft_cfm := cfm(spring, damping, fixed_step)
    contact.friction := material_a.friction · material_b.friction

    # --- 3. non-solid material effects --------------------------------------
    FOR EACH side (a, b) WHERE that side IS triangle_soup
      IF side.material HAS slow_down AND the opposite side is not already being pushed out THEN
        body := the opposite side's body                 # static-versus-static is a hard error
        IF side.material IS liquid
           OR the opposite object does not trace its motion THEN
          accumulate_contact_effector(body, contact, side.material)
      IF side.material HAS passable THEN do_collide := false

    # --- 4. bounce, only when BOTH surfaces allow it ------------------------
    IF material_a HAS bounceable AND material_b HAS bounceable THEN
      contact.mode := contact.mode | bounce
      contact.bounce_threshold := MAX(material_a.bounce_start_velocity,
                                      material_b.bounce_start_velocity)
      contact.restitution      := MIN(material_a.bounce, material_b.bounce)

    # --- 5. per-object callbacks may amend or veto --------------------------
    IF data_b HAS callbacks THEN data_b.callbacks(INOUT do_collide, is_first=false, contact, ...)
    IF data_a HAS callbacks THEN data_a.callbacks(INOUT do_collide, is_first=true,  contact, ...)

    # --- 6. depenetration override ------------------------------------------
    FOR EACH side WHERE that side records a push-out state
      clear that state if the triangle it refers to is passable
      pushing := pushing OR that side's push-out state
      IF that side belongs to a physics object THEN
        object.init_contact(INOUT contact, INOUT do_collide, material_a, material_b)
    IF pushing THEN contact.friction := UNBOUNDED

    # --- 7. commit ----------------------------------------------------------
    IF do_collide AND committed < max_contacts THEN
      committed := committed + 1
      constraint := new contact constraint from contact, drawn from ContactGroup
      island.attach_for_this_step(constraint)
      constraint.connect(body_of(shape_a), body_of(shape_b))

  RETURN committed
```

**Invariants** — a contact between two shapes that both lack a body is a hard error: the collider
should never have been asked for a static-versus-static pair, and the collision-group filter is
what guarantees it.

**Notes on the decisions in that loop, in order of how much they matter:**

*Material parameters are pairwise products, not lookups.* Friction, spring and damping are each the
product of the two surfaces' own factors and the world's baseline. This is why a single surface's
"friction" value in the material data is a dimensionless multiplier rather than a coefficient: the
number only means something against another surface. Bounce is the exception and uses `min` for the
restitution and `max` for the threshold — the *least* bouncy surface wins, and it takes the
*faster* of the two impact thresholds to start bouncing at all. That asymmetry is deliberate: a
rubber ball should bounce off stone but not off mud, and the conservative choice in each field
gives that.

*Passability is a veto, not a filter.* A surface marked passable still produces contact points, and
those points are still handed to `init_contact` — the veto happens after. That is what lets the
character controller learn "my foot is in tall grass" and pick up the grass's material for
footstep sound and injury without the grass stopping it. See `foot_material_update` in
[`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md).

*Slow-down and liquid are accumulated, not applied.* A "slow down" material does not adjust the
contact; it registers a *contact effector* on the body, and several contacts against the same
material merge into one effector, applied once in the tune phase. Applying per contact would make
the drag depend on how many triangles happened to be touched — a wide puddle would slow you more
than a narrow one of the same depth.

*Push-out sets infinite friction.* When the depenetration machinery has decided a body must be
pushed out of geometry it is inside, friction at that contact becomes unbounded so the body cannot
slide along the surface it is escaping. Without it, an object squeezed into a wall slides along the
wall forever instead of popping out.

*The callback order is b-then-a.* Both sides get to veto, and the second caller sees the first
caller's edits. The engine relies on this in at least one place (the character's foot handling
assumes it runs after the generic object callback).

*Truncation is silent.* If the collider returns more points than the scratch buffer holds, or more
than the island's remaining constraint budget allows, the excess is dropped without a word. The
budget comes from [`PHIsland.h`](PHIsland.h.md) and is the frame-time safety valve; the buffer
limit is just a buffer limit. A rebuild should at minimum distinguish the two, because one is a
policy and the other is a bug waiting to happen.

## `NearCallback`

**Contract** — the pairwise entry point. Resolves both objects' live islands, notifies the second
object that a neighbour is near, checks the merge budget, generates contacts, and — only if any
were produced — merges the islands and wakes the passive partner.

```text
FUNCTION near_callback(object_a, object_b, shape_a, shape_b)
  island_a := live_island_of(object_a)
  island_b := live_island_of(object_b)
  object_b.near_callback(object_a)                 # proximity notification, independent of contact
  (allowed, budget) := island_a.can_merge(island_b)
  IF NOT allowed THEN RETURN                       # over the island size limit: skip this pair
  IF collide_into_group(shape_a, shape_b, island_a, budget) > 0 THEN
    object_a.merge_island(object_b)
    IF NOT object_b.is_active THEN object_b.wake(object_a)
```

**Notes** — the proximity notification fires whether or not a contact results, which is how a
climbable ladder tells a character it is nearby before the character touches it (see
[`PHStaticGeomShell.cpp`](PHStaticGeomShell.cpp.md)).

Merging only *after* a contact exists is what keeps islands minimal: two objects whose bounding
boxes overlap but whose shapes do not stay in separate islands and solve separately.

## `CollideStatic`

**Contract** — collide one object against the level's triangle soup, into that object's island,
with the island's full remaining constraint budget.

## `BodyCutForce`

**Contract** — clamps the force and torque already accumulated on a body so that neither can
produce more than the given velocity change in one step. Mutates the body's pending force and
torque in place; allocates nothing.

```text
FUNCTION body_cut_force(body, linear_limit, angular_limit)
  # linear: a force may not change speed by more than linear_limit in one step
  force_ceiling := linear_limit / fixed_step · body.mass
  IF |force| > force_ceiling THEN force := force · (force_ceiling / |force|)

  # angular: convert torque to angular acceleration through the GLOBAL inertia tensor,
  # clamp there, and convert back
  IF |torque| < tiny THEN RETURN
  I        := rotate_to_world(body.inertia,         body.orientation)
  I_inv    := rotate_to_world(body.inverse_inertia, body.orientation)
  accel    := I_inv · torque
  ceiling  := angular_limit / fixed_step
  IF |accel| > ceiling THEN torque := I · (accel · ceiling / |accel|)
```

**Notes** — the clamp is on *acceleration*, not force, which is why the inertia round trip is
necessary. A long thin object and a compact one of the same mass need very different torques to
spin at the same rate; clamping the torque directly would make one of them unusable. This is called
after every single externally-applied impulse — a bullet hit, an explosion, a script push — and is
the reason a grenade blast does not launch a crate into orbit.

## `mass_sub`

**Contract** — subtracts mass distribution `b` from `a` in place: total mass, centre of mass and
inertia tensor. The inverse of the library's mass-add.

```text
FUNCTION mass_sub(INOUT a, b)
  inv := 1 / (a.mass - b.mass)
  a.center  := (a.center · a.mass - b.center · b.mass) · inv
  a.mass    := a.mass - b.mass
  a.inertia := a.inertia - b.inertia
```

**Notes** — exists solely for breakables. When a piece separates from a body, the remainder's mass
must be the original minus the piece's, and the dynamics library only offers addition. The
subtraction is exact only while both tensors are expressed about the same reference point, which is
a precondition the breakable code maintains by construction (see
[`PHFracture.cpp`](PHFracture.cpp.md)) and which a rebuild must preserve.

## `E_NlS` / `E_NLD` / `E_NL` — inelastic collision energy

**Contract** — the kinetic energy that would be lost if the collision along a given normal were
perfectly inelastic. Three cases: body against immovable, body against body, and a dispatcher.
Returns zero when the pair is separating rather than approaching.

```text
FUNCTION energy_body_vs_static(body, normal, sign) -> real
  approach := MAX(0, -dot(body.velocity, normal) · sign)
  RETURN body.mass · approach² / 2

FUNCTION energy_body_vs_body(a, b, normal) -> real       # normal points from b toward a
  va := dot(a.velocity, normal) ; vb := dot(b.velocity, normal)
  IF va > vb THEN RETURN 0                               # separating
  combined_mass     := a.mass + b.mass
  centre_of_mass_v  := dot((a.momentum + b.momentum) / combined_mass, normal)
  before := va²·a.mass/2 + vb²·b.mass/2
  after  := centre_of_mass_v² · combined_mass / 2        # both moving together
  RETURN before - after
```

**Notes** — this is the *physically available* energy of an impact, used as the basis for impact
damage and for deciding whether a hit is worth a sound or a decal. Computing it from the
centre-of-mass frame rather than from relative velocity makes the two-body and one-body cases agree
in the limit as one mass grows, which matters because the same damage curve is applied to "hit by a
crate" and "hit the ground".

## `FixBody` / `apply_gravity_accel` / `contact_position`

**Contract** — `FixBody` makes a body effectively immovable (see the note in
[`Physics.h`](Physics.h.md) about why this is done with numbers instead of a static-body flag).
`apply_gravity_accel` turns an acceleration into a mass-scaled force so that a per-object gravity
override behaves like gravity rather than like a push. `contact_position` reads a contact
constraint's world-space point back out, asserting that the constraint really is a contact.
