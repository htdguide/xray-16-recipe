# src/xrPhysics/PHSimpleCharacter.cpp

> The character controller: how a capsule that is told where it wants to go decides whether it is standing, falling, climbing or being thrown, and rewrites its own contacts every step to make walking feel like walking.

**Needs** — [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`ElevatorState.h`](ElevatorState.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHDisabling.h`](PHDisabling.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`CalculateTriangle.h`](CalculateTriangle.h.md) · [`SpaceUtils.h`](SpaceUtils.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`DamageSource.h`](DamageSource.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`ph_valid_ode.h`](ph_valid_ode.h.md) · [`tri-colliderknoopc/__aabb_tri.h`](tri-colliderknoopc/__aabb_tri.h.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md)
**Tier floor** — T1: it edits solver contacts in place during the collision phase and runs box queries against the collision database every step.

## Purpose

**A character is not a rigid body, and this file is the proof.** A rigid body is told what forces act
on it; a character is told where it wants to go. Turning the second into the first, in a way that
feels like walking rather than like pushing a barrel, is the whole content of the chapter's hardest
file — and almost none of it is physics. It is a state machine over contacts, plus a set of
deliberate lies told to the solver.

Four things happen here and they are independent:

1. **Deciding what the world is doing to the character** — the contact callback, which draws a
   ground, a wall, a friction and a material out of a pile of contact points.
2. **Deciding what state the character is in** — standing, falling, climbing, jumping, thrown.
3. **Turning the intent into force** — the control force, with its surface-following and its
   side-slip cancellation.
4. **Measuring what the collisions should cost in damage.**

## `InitContact` — rewriting the world's answer

**Contract** — called once per candidate contact during the collision phase, before the solve, with
the ability to modify the contact or reject it. This is the seam the
[rigid-body seam](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) calls "the engine injects
its own contacts between collision detection and the constraint solve", and the character is its
heaviest user.

```text
FUNCTION init_contact(INOUT contact, INOUT do_collide, material_1, material_2)
  bo1 := this character owns the contact's FIRST shape
  material := the OTHER side's material

  IF climbing AND under control THEN
    friction := 0 ; make the contact soft ; record contact          # a ladder is frictionless

  IF the material is PASSABLE and the contact was already rejected THEN
    account the damage ; update the foot material ; RETURN          # grass, water, bushes

  IF do_collide THEN record that a contact happened

  foot_process(contact, do_collide, bo1)                            # the step-climbing lie
  IF the contact was rejected THEN RETURN

  IF the contact is on the HEAD shape THEN
    side_contact := true ; soften it ; friction := 0

  IF the other side is a BODY (not the level) THEN
    stiffen the contact tenfold
    account dynamic collision damage
    IF the contact is on the FOOT sphere THEN take the foot material from that object

  update the foot material
  friction_factor := MAX(friction_factor, this contact's friction)
  contact_count := contact_count + 1

  # choose THE ground and THE wall out of every contact this step
  IF this contact's normal is more upward than the stored ground normal THEN
    ground_normal, ground_position := this contact's
  IF this contact's normal opposes the intent more than the stored wall normal THEN
    wall_normal, wall_position := this contact's

  soft_param := damping + normal.up · (1 - damping)
  IF under control THEN
    friction := 0                            # THE BIG LIE, see below
    soften the contact by (spring, soft_param)
  ELSE
    soften the contact by (spring, soft_param)
    friction := friction · (1 + 3·climbing) · friction_factor
  account static collision damage
```

**Invariants** — a contact's normal is stored in *this character's* sense. When the character owns
the second shape the normal points the wrong way and is inverted on the way in. Getting that
backwards makes a character treat floors as ceilings.

**Notes** — the decisions, in order of how much they matter.

***Friction goes to zero while the character is moving under its own control.*** This is the single
most important line in the controller and it looks like a bug. The reasoning: a character's
propulsion is an applied force, not a friction-mediated push-off. If the ground also had friction,
the force would have to overcome it, the character would accelerate sluggishly and — worse — would
accelerate *differently* on different surfaces, which the movement tuning cannot accommodate. So
while under control, the ground is frictionless and the controller supplies all the resistance
itself, through the side-slip cancellation in `PhTune`. When control stops, friction comes back and
the character slides to a halt naturally. A rebuild that keeps real friction must replace the whole
propulsion model.

***Contacts are made soft, and the softness depends on the surface's tilt.*** A perfectly rigid
contact makes a character land like a dropped stone and jitter on slopes; a soft one lets the solver
resolve penetration over several steps. The damping is blended toward one as the surface tilts away
from horizontal, so a wall is stiffer than a floor — which is right, because a soft wall is a wall
you sink into.

***Contacts against dynamic bodies are ten times stiffer.*** A character standing on a crate would
otherwise sink into it, because the crate is also yielding.

***The head shape sets `side_contact` and takes zero friction.*** A character whose head is against
something must not be allowed to climb it, and must not be able to hang from it by friction.

***Passable materials are accounted and then abandoned.*** Grass, water and foliage reject the
contact upstream; the character still reads their material as what it is standing in, and still
takes fall damage through them at a reduced rate. This is how wading works.

## `FootProcess` — the step-climbing lie

**Contract** — rewrites a contact's normal and penetration depth so that a low obstacle reads as
flat ground under the character's feet.

```text
FUNCTION foot_process(INOUT contact, INOUT do_collide, bo)
  sign := +1 if this character owns the first shape, else -1

  IF climbing, or not under control, or not clambing, or any side contact,
     or jumping THEN
    IF the contact is on the FOOT sphere AND its normal points DOWNWARD THEN
      do_collide := false                      # never let the foot be pushed down
    RETURN

  IF the contact opposes the intended direction THEN RETURN     # a real wall: leave it alone

  height := contact.position.up - body.position.up
  IF the contact is on the FOOT sphere AND its normal is not upward THEN
    normal := straight up ; depth := height + radius
  IF the contact is on the BODY CYLINDER AND the contact is BELOW the body origin THEN
    normal := straight up ; depth := height + radius
```

**Invariants** — the rewrite happens only when the character is actively moving into the obstacle and
the obstacle is not classified as a wall. That is the whole safety of it: a real wall's contact
opposes the intent, is detected by the first test, and is never rewritten.

**Notes** — this is the mechanism that lets a character walk up a kerb, a stair tread or a low rock
without a jump, and it is worth understanding as a *lie told to the solver*. The solver sees a
contact whose normal points straight up and whose penetration is the height of the obstacle; it
responds by pushing the character straight up by that amount. There is no climbing animation, no
raycast-and-teleport, no separate stepping state — just a contact normal bent to vertical.

The height limit is implicit: the depth is computed against the foot sphere's radius, so an obstacle
taller than the foot sphere produces a contact on the cylinder above the body origin, which is not
rewritten. **The maximum step height is therefore the character's own radius**, and it changes if
the character is resized. That is a real constraint a rebuild inherits: the shape model and the step
height are the same number.

The unconditional rule at the top — a downward-pointing contact on the foot sphere is always
rejected — is the defence against being pressed into the floor by something above.

## `PhTune` — the state machine

**Contract** — pre-solve, once per step. Reads the contacts the previous collision phase gathered,
derives the controller's state, and applies the control force. This is the largest procedure in the
chapter and it is a sequence of independent decisions, not an algorithm.

```text
FUNCTION ph_tune(step)
  remember the body's position for the smoothed-velocity readout
  advance the climbing state machine
  air_contact_state := NOT is_contact

  good_ground := valide_ground_contact AND ground_normal.up > cos(45°)

  # 1. the "being pushed out of geometry" latch
  IF the foot shape is being pushed negatively THEN
    death_position := where the body was one step ago       # a known-safe place
  IF the pushing stopped THEN clear the latch

  # 2. apply any accumulated contact effector (liquid drag, slow-down fields)

  IF the body is asleep THEN clear lose_control and RETURN

  # 3. the edges
  is_control  := |acceleration| > 0.1
  depart      := was_contact AND NOT is_contact
  meet        := NOT was_contact AND is_contact
  stop_control:= was_control AND NOT is_control
  IF lose_control AND (is_contact OR climbing) THEN meet_control := true   # one-shot latch
  on_ground   := valide_ground_contact OR (meet AND NOT depart)

  # 4. climbing overrides
  IF climbing THEN
    side_contact := false ; friction_factor := 1
    IF control just stopped THEN zero the velocity          # a ladder holds you

  # 5. regaining control
  IF depart THEN remember where the character left the ground
  IF lose_control AND ( (on_ground AND ground is not too steep)
                        OR the character is motionless
                        OR climbing ) THEN lose_control := false

  # 6. ending a jump
  IF (jumping AND good_ground) OR (climbing AND a wall contact) THEN jumping := false

  # 7. losing control by flying
  IF NOT on_ground AND NOT climbing THEN
    IF the character has moved more than half a metre from where it left the ground
       AND at least a tenth of that vertically THEN
      lose_control := true ; depart_control := true

  validate_walk_on()                                       # the step/ledge decision

  # 8. starting a jump
  IF jump requested THEN
    lose_control := true ; depart_control := true
    velocity := the jump velocity OUTRIGHT
    remember the departure position ; jumping := true ; leave any ladder ; wake up

  lose_ground := NOT (good_ground OR climbing) OR lose_control

  apply_acceleration()                                     # builds the control force

  # 9. the propulsion itself
  IF is_control THEN
    side := the horizontal direction perpendicular to the control force
    apply the control force
    IF NOT lose_control OR clambing THEN
      cancel the component of velocity along `side`, at 500 (or 700 while clambing)
        times the friction factor
      IF in control AND airborne THEN also press DOWN at 50 times the mass

  # 10. air control while jumping
  IF jumping THEN
    factor := 1, or ten times the air-control factor for the player while ballistic
    IF the character has not yet moved 0.3 in the intended direction THEN
      apply a horizontal force of 1000 · factor along the intent
    IF the velocity opposes the intent THEN
      apply a horizontal braking force of 3000 · factor scaled by how much it opposes

  clamp the accumulated force and torque
```

**Notes** — the decisions worth naming.

***Losing control is a distance test, not a time or contact test.*** A character is considered
ballistic once it has travelled half a metre from wherever it last touched ground, *and* at least a
tenth of a metre of that was vertical. The vertical clause is what stops a character from going
ballistic while walking off the edge of a ramp, and the distance is what stops it from flickering on
every bump. The two thresholds are tuning; there is no derivation.

***Regaining control requires only a not-too-steep ground contact.*** The slope test here uses a
*gentler* limit than the one that defines good ground — a shallower requirement to land than to
stand. That asymmetry means a character can regain control on a slope it will then slide down,
which is intentional: it is how landing on a hillside looks controlled rather than like a tumble.

***Side-slip cancellation is the friction the controller supplies itself.*** Because the ground was
made frictionless in the contact callback, nothing stops a moving character from sliding sideways.
The remedy is a force proportional to the velocity component perpendicular to the intended
direction, with a large coefficient. That is a damper, not a friction model — it has no static
component and cannot hold a stationary character on a slope, which is why a character that stops
moving gets its friction back.

***The downward press while under control and airborne.*** Fifty times the mass, downward, whenever
the character is trying to move and is not touching anything. This is what keeps a running character
glued to the ground over crests and down slopes instead of launching off them. It is a large force
and it is why a character cannot run off a ramp and fly.

***A jump sets the velocity outright*** rather than applying an impulse, because a jump must reach
the same height regardless of what the character was doing — running, standing, or being pushed.

***Air control is ten times stronger for the player than for a creature***, and only while ballistic.
This is a game-feel decision, not a physical one: the player expects to steer in the air and a
creature does not.

## `ApplyAcceleration` — intent to force

**Contract** — builds the control force from the intent, the surface underfoot and the state.

```text
FUNCTION apply_acceleration()
  control_force := 0
  IF lose_control THEN
    control_force := acceleration · mass · air_control_factor
    RETURN                                            # ballistic: only air control applies

  accel := the intent, flattened to horizontal
  IF on a ladder THEN take the direction from the ladder instead; while climbing,
     the force is the ladder's direction · friction · mass · 2·pull_force and we are done

  side := horizontal perpendicular to the intent
  IF clambing against a wall THEN
    direction := the wall's surface direction     # push ALONG the wall, i.e. up it
  ELSE IF the ground is not too steep THEN
    direction := the ground's surface direction   # follow the slope
  ELSE
    direction := the flat intent, at 1.5× strength   # no usable surface: push harder
  control_force := direction · mass · pull_force

  IF clambing and not on a ladder THEN
    scale the force fourfold
    force the vertical component UPWARD
    force the horizontal components to agree in sign with the intent
  scale by the friction factor
```

**Invariants** — the force is always proportional to **mass**, so a heavy creature and a light one
accelerate alike. `pull_force` is 25 — an acceleration in units of gravity-and-a-half, which is the
number that makes walking look like walking and is pure tuning.

**Notes** — *following the surface* is the load-bearing idea. The intent is horizontal; the force is
the intent rotated into the plane of the ground. Without it, walking up a slope would push into the
hill rather than along it, and the character would climb only as fast as the contact solve let it
slide.

The steep-ground fallback pushing *harder* is deliberate: with no usable surface the horizontal push
is partly wasted against the contact, so it is compensated.

The clamb multiplier — four times the force, forced upward, sign-corrected — is not physics. It is
the amount of shove needed to get over a ledge whose contact has been bent vertical by `FootProcess`,
found by tuning.

While the intent is scaled by the air-control factor during ballistic flight, a factor of zero — the
default for creatures — means a thrown creature is completely unsteerable.

## `ValidateWalkOnMesh` — may the character climb here?

**Contract** — a per-step query against the static collision database that answers: is there
something ahead worth climbing onto, and is there something above that forbids it? Sets the
side-contact flag as a side effect. Returns whether climbing is permitted.

```text
FUNCTION validate_walk_on_mesh() -> bool
  IF there is no intent THEN RETURN true            # not moving: nothing to forbid
  ahead := the foot position, shifted 0.4 along the intent

  allow_box  := a box of the character's radius, centred at `ahead`, one radius up
  forbid_box := the same box, doubled in height, centred 1.5 above `ahead`

  ONE box query covering the union of both boxes

  # pass 1 — the forbidding box, checked first
  FOR EACH triangle hit
    skip PASSABLE materials
    IF the triangle overlaps the forbid box AND the character's oriented box meets it THEN
      side_contact := true
      RETURN false                                   # something is at head height: no climbing

  # pass 2 — the allowing box
  FOR EACH triangle hit
    skip PASSABLE materials
    IF the triangle overlaps the allow box AND the character's oriented box meets it THEN
      RETURN true                                    # something climbable is there
  RETURN false
```

**Invariants** — **one query, two uses.** The two boxes are merged into a single query against the
collision database, and the results filtered twice. A rebuild must not issue two queries; this runs
every step for every active character and the query is the expensive part.

**Notes** — the forbidding pass runs first and returns immediately, so an overhang always wins over a
climbable ledge. That is what stops a character from climbing into a low ceiling and becoming stuck.

The oriented box test (`test_sides`) is a separating-axis test between the character's box — oriented
along its intended direction — and a triangle, checked against the triangle's normal, its most
significant edge direction, and the three edge-cross axes. The broad-phase box test that precedes it
is axis-aligned and admits far too much; the oriented test is what makes the answer precise enough
that a character climbs a step in front of it and not one beside it. A rebuild may use any
triangle-versus-oriented-box test; what must be preserved is that the box is oriented along the
*intent*, not along the world axes, because the whole question is "is the thing I am walking into
climbable".

The offsets — 0.4 ahead, 0.05 above the feet, 1.5 up for the forbidding box, and a box of half the
radius by the radius by seven tenths of the radius — are tuning constants with no derivation. They
encode how far ahead a character looks and how much headroom it demands.

## `ValidateWalkOnObject`

**Contract** — the same decision for *dynamic* objects, made from the contacts already gathered
rather than a query.

```text
FUNCTION validate_walk_on_object()
  IF already clambing AND the character has risen more than half a metre since starting
    THEN stop clambing                              # a clamb has a maximum height

  IF not on a ladder
     AND there is a wall contact AND more than one contact
     AND the wall is steeper than 45°
     AND there is no side contact THEN
    IF the wall contact is AHEAD of the ground contact in the direction of the control force
       AND the wall contact is ABOVE the ground contact THEN
      start clambing

  IF still clambing THEN remember the current position as the clamb's starting height
```

**Notes** — the geometric test is the interesting part: a *step* is a contact that is both ahead of
where the feet are and higher than them. A wall that is ahead but not higher is a wall; a contact
that is higher but not ahead is a ceiling. Requiring more than one contact means the character must
be touching both the ground and the obstacle — climbing from mid-air is not allowed.

The half-metre travel limit is what ends a clamb. Without it a character that is clambing against a
tall wall would keep being pushed up it indefinitely.

## `SafeAndLimitVelocity` — the speed cap and the recovery

**Contract** — post-solve, first thing. Clamps the character's speed to a limit that depends on its
state, and — separately — recovers from a non-finite solver result.

```text
FUNCTION safe_and_limit_velocity()
  IF the velocity is NOT a finite vector THEN
    restore the last known-good velocity
  ELSE
    limit := (under control and not ballistic) ? max_velocity / time_factor
                                               : the world's default limit
    IF an external impulse is active THEN raise the limit to let the impulse through
    IF speed > limit THEN
      IF falling with no ground and descending slowly THEN
        scale only the HORIZONTAL velocity        # do not fight gravity
      ELSE
        cut the velocity (advancing the body by the removed amount first)
      IF under control THEN
        re-place the body one step's travel from the last known-good position
  IF the position is NOT finite THEN restore it from the last known-good position and velocity
  record the position and velocity as the new known-good pair
```

**Invariants** — a known-good position and velocity are recorded **every step**, unconditionally, and
are the recovery source. This is the character's version of the
[determinism and validity requirement](../../SYSTEM-REQUIREMENTS.md#6-conformance): a character
whose state goes non-finite must return to a sane one rather than propagate the failure.

**Notes** — the speed cap is different in the two states and that is the point. Under control, the
cap is the *game's* walking speed, divided by a global time factor so that a slowed or accelerated
game still walks at the right apparent speed. Ballistic, the cap is the world's generic safety limit,
which is far higher — a falling character must be allowed to fall fast.

The falling special case scales only the horizontal component, because clamping total speed while
falling would make a long fall slow down, which looks wrong and breaks fall damage.

The re-placement after a clamp is the same correction as the rigid body's: the solver has already
integrated at the *unclamped* velocity, so the body must be put back where the clamped velocity would
have carried it. Here it is done from the known-good position rather than by integrating backwards.

`mean_y` is a very slowly-moving average of vertical velocity, updated every step with a weight of
one ten-thousandth. Nothing in this file reads it.

## `PhDataUpdate` — the post-solve pass

**Contract** — clamp and recover, decide about sleeping, expire an external impulse, clear the step's
conclusions, enforce the upright constraint, apply air drag, and push an interpolation sample.

```text
FUNCTION ph_data_update(step)
  safe_and_limit_velocity()
  IF asleep THEN clear lose_control and RETURN
  IF touching something AND not being controlled AND standing on good ground THEN
    run the sleep accumulator

  IF an external impulse has expired THEN clear it and ZERO the velocity

  copy every is_* conclusion into its was_* counterpart, then clear them all
  # the upright constraint
  angular velocity := 0 ; orientation := identity
  apply quadratic air drag, capped so it cannot reverse the velocity in one step
  IF non-interactive THEN sleep and hold position
  record this step's displacement as the smoothed velocity
  IF outside the world's boundaries THEN sleep
  push an interpolation sample
```

**Invariants** — the **upright constraint** is applied here and unconditionally: a character's
orientation is reset to identity and its angular velocity zeroed every single step. Its facing
direction is not in its physics state at all; it lives in the game object. A rebuild that lets the
capsule tumble has built a ragdoll.

**Notes** — sleep requires all three of touching something, not being asked to move, and having good
ground. A character standing on a slope it is sliding down never sleeps, which is correct.

The expiry of an external impulse **zeroes the velocity outright**. A character thrown by an
explosion travels for a fixed number of steps and then stops dead. That is visibly artificial and it
is what the shipped game does; the alternative — letting the impulse decay — made characters slide
for long distances.

The air drag is the same quadratic model and the same one-step cap as the rigid body's; see
[`PHElement.cpp`](PHElement.cpp.md).

## `ApplyImpulse` — being thrown

**Contract** — replaces the character's velocity with a large force applied over a fixed number of
steps, and puts it into the ballistic state for the duration. Refuses while another impulse is
active.

```text
FUNCTION apply_impulse(direction, magnitude)
  IF already carrying an impulse THEN RETURN
  IF the character is already airborne or jumping THEN direction := straight DOWN
  wake up ; lose_control := true
  expire_at := current_step + 30
  velocity := 0
  force := direction · magnitude / fixed_step
```

**Notes** — the airborne override is the striking decision: **an explosion that hits a character who
is already in the air slams them downward instead of pushing them along the blast direction.** A
character in flight has no contact to absorb a second push, and applying one produces the
long-distance launches that this replaces. Slamming them down gets them back to the ground where the
next impulse will behave. It is a game-feel rule, not physics, and it is deliberate.

Thirty steps is the impulse's lifetime and it is a bare constant. At the engine's fixed step that is
a fraction of a second.

## collision damage

**Contract** — every contact is offered to the damage accounting, which keeps only the most severe of
the step. Static and dynamic collisions use different models; both are in
[`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md).

**`ContactBone`** answers *where* the blow landed, which the damage layer needs to pick a body part.
It casts a short ray along the contact normal against the character's own collision model and takes
the bone it hits; failing that, it falls back to the nearest bone by distance in object space.

**Notes** — the fallback exists because a contact can be generated against the character's *physics*
capsule at a point the render model's collision does not cover. The nearest-bone search is a linear
scan over every bone, which is cheap enough because it only runs when the ray missed.

The damage initiator is resolved through the *hitting object's own* damage-source chain: a crate
thrown by the player credits the player, not the crate. A static collision credits the character
itself, which is how falling damage is attributed.

## `UpdateRestrictionType` — resizing a character without overlapping another

**Contract** — change a character's mutual-exclusion size class, but only if the new size does not
leave it overlapping another character. Iteratively steps the world with everything else frozen
until the overlap resolves, or gives up and reverts.

```text
FUNCTION update_restriction_type(other_character) -> bool
  IF the type is already current THEN RETURN true
  wake both characters ; freeze the world
  adopt the new size ; install a probe contact callback that records the deepest overlap
  unfreeze this character ; run one collision-only pass
  IF the deepest overlap is under 5 cm THEN unwind and RETURN true

  steps := 2 · CEILING(deepest_overlap / 5 cm)
  REPEAT steps TIMES
    run one full world step with only these two characters live
    IF the deepest overlap is under 5 cm THEN unwind and RETURN true
  revert to the old size ; unwind ; RETURN false
```

**Invariants** — the world is frozen for the whole procedure and unfrozen on every exit path. The
probe callback is installed and removed in the same scope. A failure **reverts the requested type**,
so a character that cannot grow stays small.

**Notes** — the size classes are not collision shapes; they are mutual-exclusion volumes that keep
creatures of different sizes at sensible distances (see
[`PHCharacter.h`](PHCharacter.h.md)). Growing into a space another creature occupies must be refused
rather than resolved by force, because forcing it launches both. Stepping the world with everything
else frozen is how the engine gives the two characters a few steps to push apart on their own without
the rest of the level advancing — a local, bounded, reversible simulation. The step budget is
proportional to the overlap, so a small overlap is cheap and a large one is refused quickly.

The direction is asymmetric by construction: shrinking always succeeds, growing may fail. The debug
message says so.

## material under the feet

**Contract** — what the character is standing on, which drives footstep sounds, damage and the
injury from hazardous surfaces. Two sources: the contacts themselves, and a downward ray.

**`update_last_material`** casts a ray half a metre down from just above the feet, skipping materials
flagged as actor obstacles, and caches the result against the position it was taken at. A move of
less than a tenth of a metre reuses the cached answer.

**`foot_material_update`** (in [`PHSimpleCharacterInline.h`](PHSimpleCharacterInline.h.md)) is the
contact-driven half, and encodes the rule that a *passable* material — water, grass — is what you are
standing **in**, while the solid surface under it is what you are standing **on**.

**Notes** — the ray exists because contacts do not always reach the ground: a character standing on
grass collides with the terrain underneath, and the grass is what should be heard. Caching by
position rather than by time is right for a character that is often standing still.

An *injurious* material is latched separately and cleared at the start of each collision phase, so
that standing in acid damages per step and stepping out stops it immediately.

## the four shapes, created

**`Create`** builds the foot sphere, the body cylinder, the head sphere and the path probe; binds the
first three to the body and puts all four in the character's own collision space; gives the body a
mass with an enormous inertia tensor; and registers the character with the collision filter as a
character.

**Notes** — the **enormous inertia** is how the upright constraint is enforced *in the solver* as well
as after it. A body whose inertia is a million times its mass is effectively unrotatable by any
contact, so the solver produces no angular motion to correct. The explicit reset in the post-solve
pass is the belt to that braces. A rebuild whose solver can lock rotational degrees of freedom should
do that instead and skip both.

The path probe is told **not to collide with the static world** and carries a callback whose entire
body sets the side-contact flag. It is a sensor: it answers "is there a dynamic object where my head
is about to be", which the static-geometry query cannot.

**`SetBox`** resizes the four shapes in place from a bounding box, deriving the radius from the
smaller horizontal extent and the cylinder height from what is left after the two spheres. A box
whose height is less than its width yields a degenerate cylinder height, which is clamped to a
hundredth of a metre rather than refused.

## `Destroy`

**Contract** — refuses if the world is mid-step or frozen; deactivates the ladder state, unregisters
spatially, and destroys the four shapes, their transforms, the space and the body in that order.

**Invariants** — shapes before space before body. A shape still in a space when the space is
destroyed, or still bound to a destroyed body, is the classic crash.

## positions — three of them, and they differ

**Contract** — the character reports three different points and confusing them is a real bug:

- **`GetPosition`** — the point on the ground under the character: the body's origin lowered by the
  foot radius. This is what the game means by "where the character is".
- **`GetBodyPosition`** — the body's own origin, at the centre of the foot sphere.
- **`IPosition`** — the interpolated ground position, for rendering.

**`DeathPosition`** is a fourth: the last position known to be *outside* geometry, held while the
character is being pushed out of something. A corpse is placed there rather than where the body
currently is, so that bodies do not end up inside walls.

**`GetSmothedVelocity`** reports the displacement of the last step divided by the step, not the
body's velocity. The difference matters: the solver's velocity includes what the contacts are about
to undo, and animation driven by it flickers.

## `CheckInvironment`

**Contract** — the controller's published summary, which the animation layer branches on: ballistic
means *in air*, climbing means *at a wall*, anything else means *on ground*.

**Notes** — the summary is deliberately coarser than the internal state. In particular a character
that is clambing over a step reports itself on the ground, because there is no climbing animation for
a step — the whole point of the step-climbing lie is that it is invisible.
