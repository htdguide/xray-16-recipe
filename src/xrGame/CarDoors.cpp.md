# src/xrGame/CarDoors.cpp

> A vehicle door as a motor-driven hinge with five states, and the harder half: the geometry that answers "can a person get in or out through this doorway".

**Needs** — [`Car.h`](Car.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`Hit.h`](Hit.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it activates and deactivates joints inside a live physics assembly and reads joint anchors and axes back out

## Purpose

Two separate problems share this file.

**A door is a hinge with a motor**, and the interesting part is that its joint is
*deactivated while closed*. A closed door is a rigidly skinned bone like any other; only
when it starts moving is a physics element and a joint brought into existence for it, and
they are torn down again when it comes to rest. The reason is cost: a shipped level can
hold a dozen vehicles with four doors each, and forty-eight always-solved hinge joints buy
nothing while every one of them is shut.

**A doorway is a hole**, and deciding whether a person fits through it is not the same
question as whether the door is open. The second half of the file builds a plane with two
extents out of the door's own collision geometry at load time, and then tests a ray
against that plane.

The bind-transform scratch buffer these tests use is a file-scope singleton. That is an
allocation choice, not a design one; a rebuild allocates per call or per thread.

## State

```text
RECORD Door                       # nested in the vehicle; see Car.h
  bone_id             : bone
  joint               : optional<hinge>      # none for a "fake" door that only looks like one
  state               : ENUM { opening, closing, opened, closed, broken }
  torque              : real      # derived at init from the leaf's mass and lever arm
  a_vel               : real      # opening angular speed; defaults to half a turn per second
  pos_open            : real      # +1 or -1: which way around the hinge "open" is
  opened_angle        : real      # the two stops, ordered so that pos_open * angle increases
  closed_angle        : real      #   as the door opens, whichever sign pos_open has
  open_time           : int       # when the door reached "opened"
  update              : bool      # is this door currently in the vehicle's update list
  door_plane_ext      : (real, real)   # the doorway's two extents
  door_plane_axes     : (int, int)     # which local axes those extents lie along
  door_dir_in_door    : vector         # the door's outward direction, in its own frame
  closed_door_form_in_object : transform   # the door's pose when shut, in vehicle space
```

Invariants:

- `pos_open * angle` is **monotonically increasing as the door opens**, whichever physical
  direction that is. Every comparison in the state machine is written in that product, so
  one piece of code serves doors hinged either way. This is the single idea that makes the
  file short.
- a door with no joint is always answered as open. Authored models contain doorways that
  are decoration; they must still admit the player.
- `update` and membership in the vehicle's door-update list mirror each other exactly.

## `Init`

**Contract** — derive everything about a door from the physics assembly the model produced.
Runs once, after the assembly exists. Fails hard if the door's joint is not a simple hinge.

The three derivations, in order:

```text
FUNCTION init()
  joint = the hinge registered for this bone; IF none THEN RETURN (a fake door)
  REQUIRE it is a hinge
  capture the door leaf's closed pose in vehicle space

  # 1. which way is "out" for this leaf
  express the hinge axis in the leaf's own frame
  pick the leaf's LARGEST extent among the two axes that are not the hinge axis
  that axis, signed toward the leaf's longer side, is the door's outward direction

  # 2. which way around the hinge is "open"
  cast the leaf's outward direction crossed with the hinge axis against the car BODY's
  extents; open is the side with less body on it
  pos_open = +1 or -1 accordingly
  read the joint's two stops and assign them to opened_angle / closed_angle so that
  pos_open * angle grows as the door opens

  # 3. how hard the motor must push
  torque = |leaf centre - leaf centre of mass| * leaf mass * configured factor * 10

  state = opened     # the physics assembly is built with doors apart
```

**Invariants** — the stops are shrunk inward from what the model declares. The open stop is
pulled in by a quarter of its own value when the hinge opens positively, and by two degrees
when it opens negatively; the closed stop is pulled in by two degrees in the negative case.
The two cases are not symmetric and no reason survives. What the margins buy is clear
enough: a motor driven exactly to a joint stop fights the stop forever, and the state
machine would never see the door arrive.

The initial state is `opened` even though the door looks shut, because the assembly is
built from the bind pose with the joint active; the spawn path immediately forces every
door to closed.

## The state machine

### `Open`, `Close`, `Use`, `Switch`

**Contract** — the four ways a door is commanded. `Use` toggles by *intent* (a door that is
closing is going to be closed, so use it to open); `Switch` toggles only from a state at
rest. A broken door accepts nothing.

```text
FUNCTION open()
  IF no joint THEN state = opened; RETURN
  IF closed  THEN ClosedToOpening(); PlaceInUpdate()
  IF closing THEN state = opening; apply the opening motor
  # opened, opening, broken: nothing

FUNCTION close()
  IF no joint THEN state = closed; RETURN
  IF opened  THEN PlaceInUpdate()      # and fall through
  IF opened or opening THEN state = closing; apply the closing motor
```

### `Update`

**Contract** — advance one door that is in motion. Called from the vehicle's per-physics-step
hook, only for doors in the update list.

```text
FUNCTION update()
  IF closing AND pos_open * closed_angle > pos_open * current angle THEN ClosingToClosed()
  IF opening AND pos_open * opened_angle < pos_open * current angle THEN
    hold the door with a zero-speed motor at full torque
    remember the time; state = opened
  IF opened AND more than one second has passed since it opened THEN
    apply a fifth of the torque at the opening speed     # let it swing free
    leave the update list
```

**Notes** — the one-second dwell at full holding torque before releasing is what stops a
door from bouncing off its stop the instant it arrives. After it, the door is left with a
weak motor pushing it gently open, so it hangs open against gravity and against the
vehicle's acceleration instead of flapping.

### `ClosedToOpening` and `ClosingToClosed` — the joint's lifetime

**Contract** — these are the two transitions that create and destroy physics.

```text
FUNCTION closed_to_opening()
  IF the joint is already active THEN RETURN
  point the door bone's pose callback at the physics assembly's own callback
  seed the leaf element's transform from the bone's current pose
  activate the leaf element against the assembly's current global transform
  wake the assembly; activate the joint; recompute the pose

FUNCTION closing_to_closed()
  state = closed
  recompute the pose
  point the door bone's pose callback at the assembly's FIRST element, not overwriting
  deactivate the leaf element and the joint
  leave the update list
```

**Invariants** — the leaf element is seeded from the *bone's* pose, so the door starts
moving from exactly where it was drawn. On the way back, the bone is re-parented to the
body's element with overwrite disabled, which returns the door to being ordinary skinned
geometry that follows the body.

A rebuild whose physics library is cheap enough may keep every door joint alive
permanently and delete both functions. The state machine above is unchanged by that
choice; only these two are.

### `Break` and `ApplyDamage`

**Contract** — a door's damage ladder has one level, and reaching it breaks the door.

```text
FUNCTION break()
  IF closed THEN activate the joint (as for opening)
  IF opened or closing THEN leave the update list
  IF opening THEN apply a tenth of the torque with zero speed
  IF the joint exists THEN
    perturb the hinge AXIS by a small amount on all three components and renormalize
    divide the joint's spring factor by 30 and multiply its damping by 8
    do the same to the axis's own spring/damping, dividing by 20
    move the closed-side stop a quarter turn away, so it can no longer shut
  state = broken
```

**Notes** — this is a good example of a load-bearing hack. A broken door should hang
crookedly and rattle; the library has no "broken hinge", so the axis is nudged off true
(which makes the constraint slightly unsatisfiable and therefore sloppy), the constraint is
made soft and heavily damped, and the closed stop is moved out of reach. The specific
factors — 0.1 on each axis component, 1/30, ×8, 1/20, a quarter turn — are tuning with no
derivation. They are the shipped feel.

## The doorway geometry

### `TestPass`

**Contract** — does a ray from a point in a direction pass through this doorway's opening?
For a door with no joint, degenerates to "is the doorway in front of me".

```text
FUNCTION test_pass(pos, dir) -> bool
  IF no joint THEN RETURN the doorway bone is on the side of pos that dir points to
  build the closed door's plane: its anchor, its outward direction when shut,
    and the normal of those two crossed with the hinge axis
  intersect the ray with that plane
  IF the intersection is behind the ray THEN RETURN false
  RETURN the intersection lies within the leaf's extents along BOTH
         the closed outward direction and the hinge axis
```

**Invariants** — the test uses the **closed** door's plane, not the open door's. The hole a
person walks through is the hole in the car body, which does not move; the swinging leaf
only decides whether that hole is blocked. This is the load-bearing idea of the whole
second half.

### `IsInArea`

**Contract** — is a point inside the wedge swept between the closed door and where the door
currently is? Used to decide whether someone standing outside is close enough to climb in.

```text
FUNCTION is_in_area(pos, dir) -> bool
  IF no joint THEN RETURN in front, and within half the car's own half-width
  project (pos - closed anchor) onto the closed outward direction  -> a
  project it onto the current outward direction                    -> b
  project it onto each of the two plane normals and multiply them  -> c
  RETURN a and b are both positive and within the leaf's extent, and c is negative
```

**Notes** — `c < 0` is the wedge test: the point is on opposite sides of the closed door's
plane and the open door's plane, which is exactly "between them". `a` and `b` positive and
bounded keeps it within the door's own span rather than off the end of it. The whole test
is in the projections, which is why it survives the door being at any angle.

### `IsFront`

**Contract** — is this door on the side of the vehicle the approach comes from? Compares
distance-along-the-approach-axis of the door versus of the vehicle's centre, and requires
the door to be within that span sideways.

**Notes** — the approach axis is the vehicle's own lateral axis, flipped to point along the
viewer's look direction. So "front" means "the near side", not "the front of the car".

### `CanEnter`, `CanExit`, `GetExitPosition`

**Contract** — entering needs an open or broken (or fake) door, a clear pass from the foot
position, and the eye position inside the door's area. Exiting needs only a clear pass, and
is refused outright by a door that is shut.

`GetExitPosition` produces a point outside the vehicle to put the actor down on:

```text
FUNCTION exit_position() -> vector
  IF no joint THEN
    take the door bone's oriented box in world space
    drop to its lowest face along the world's vertical
    step out along its SMALLEST half-extent, away from the car's centre, times three
  ELSE
    start at the hinge anchor
    slide along the hinge axis to the leaf's lower end
    slide outward along the bisector of the closed and current outward directions,
      by the leaf's extent on the longer side
```

**Notes** — the two branches are different enough to be different ideas. The fake-door
branch lands the actor three times the door's thinnest dimension away from the car, which
is a guess that happens to clear most bodies. The real-door branch lands them at the far
lower corner of the swung-open leaf, which is where a person standing in an open car door
actually is. Neither branch tests the ground or the world for obstruction, which is why an
actor can be put down inside geometry on a bad exit — and why the explosion path is willing
to use the first door regardless.

## `DoorHit`

**Contract** — route a hit to the door on the struck bone. A melee strike above a threshold
of 20 damage **opens every door on the vehicle** before the routing.

**Notes** — that threshold is an authored feel decision in code rather than in data: hitting
a car hard enough bursts it open. It is the only place a hit affects a part other than the
one it landed on.

## `SDoorway`

**Contract** — an abandoned second implementation of the doorway extents, initialized from
the same joint data and never consulted; its trace operation is empty and its
initialization half-overwrites the door's own fields while several of its branches assign
nothing at all.

**Notes** — **not recoverable.** It is not clear whether this was meant to replace `Init`'s
axis-selection block or to sit beside it, and one of its comparisons tests an extent
against an axis component, which is dimensionally meaningless. A rebuild should omit it.
