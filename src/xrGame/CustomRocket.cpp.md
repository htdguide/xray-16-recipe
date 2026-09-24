# src/xrGame/CustomRocket.cpp

> A rocket in flight: a rigid body pushed forward by a simulated motor, trailing light, smoke and a looping sound, that stops dead at the first surface it is not allowed to pass through.

**Needs** — [`CustomRocket.h`](CustomRocket.h.md) · [`physic_item.h`](physic_item.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/PHUpdateObject.h`](../xrPhysics/PHUpdateObject.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrPhysics/CalculateTriangle.h`](../xrPhysics/CalculateTriangle.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`Include/xrRender/RenderVisual.h`](../Include/xrRender/RenderVisual.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: its contact handling runs inside the solver's collision callback and reads the library's own contact geometry

## Purpose

A rocket is the one projectile in the game that is a **simulated body** rather than a traced
bullet. It has mass, it is pushed by a motor, it drops under gravity when the motor cuts out,
and it collides. That choice buys the visible arc of an unguided rocket and costs a great
deal of care about the one moment that matters: *exactly where did it touch?*

The file holds no explosion. That is
[`ExplosiveRocket.cpp`](ExplosiveRocket.cpp.md)'s job. What this one owns is the flight: four
states, a motor, the effects that follow it, and the contact.

## State

```text
RECORD RocketState
  state           : ENUM { inactive, engine, flying, collide }
  launched        : bool
  owner           : object        # the launcher's ROOT owner, not the launcher
  launch_transform, launch_velocity, launch_angular_velocity
  time_to_explode : real          # a real-time deadline; see Notes
  contact         : { happened : bool, position : vector, up : vector }

RECORD Motor                      # absent entirely when the section says so
  present         : bool
  work_time       : int           # milliseconds the motor burns
  time_left       : int
  impulse         : real          # forward, per fixed step
  impulse_up      : real          # upward, per fixed step

RECORD Trail
  lights_enabled        : bool
  stop_lights_with_engine : bool
  trail_light           : light, with its colour and range
  flying_sound          : sound, looped
  engine_particles      : optional<effect>
  fly_particles         : optional<effect>   # must be a looping effect; asserted
  previous_velocity     : vector             # for smoothing the effects' velocity
```

Invariants:

- the state machine is one-way: inactive → engine (or straight to flying, with no motor) →
  flying → collide. Nothing returns.
- `Useful` — the reusability test for a pooled item — is true only in the inactive state.
- the rocket demands an unconditional per-frame update at all times, overriding the
  scheduler's distance-based rate reduction. A rocket updated lazily flies through walls.
- while it exists as a *child* (in the launcher), the state must be inactive; both
  attachment hooks assert it.

## `create_physic_shell` — the collision body

**Contract** — build a one-element body approximating a rocket from its visual's bounding box.
Runs once, when the rocket leaves the launcher.

```text
FUNCTION create_body()
  box = the visual's bounding box
  pick the box's LONGEST axis; that is the rocket's length
  axis   = that half-extent along that direction
  radius = the smaller of the two remaining half-extents
  halve the two remaining half-extents                      # the box narrows

  add the (now narrowed) box
  add a sphere at +axis of radius * sqrt(2)                 # the nose
  add a sphere at -axis of radius / 2                       # the tail
  mass = 7
```

**Invariants** — the composite shape is the decision: a narrowed box with a **large nose
sphere** and a small tail sphere. The nose sphere is what makes a rocket detonate on
grazing contact with a doorway rather than sliding along it, and it is deliberately larger
than the box it caps. The tail sphere keeps the body from tunnelling backwards when it is
spun.

The mass of 7 is compiled in, the same for every rocket in every game. It is not read from
data and there is no stated reason for the value.

Air resistance is set to the library default at creation and then **cleared to zero** when
the body is activated. So a rocket is unaffected by drag; its range is bounded by the motor's
burn time and by gravity alone.

## `ObjectContactCallback` — the contact

**Contract** — runs **inside the physics library's collision callback**, for every contact the
rocket generates. Decides whether this contact is the one that stops the rocket, and where.
Always suppresses the contact itself, so a rocket never bounces.

```text
FUNCTION on_contact(contact, material_a, material_b)
  do_collide = false                        # always: a rocket does not bounce off anything

  identify which side of the contact is the rocket; `up` is the other side's normal
  material = the OTHER side's material
  IF that material is marked passable THEN RETURN      # foliage, cloth, water surfaces
  IF the rocket already has a recorded contact THEN RETURN

  other = the game object on the other side
  IF other IS the rocket's owner THEN RETURN           # do not detonate on the firer

  position = the library's last recorded position for the rocket's own geometry,
             falling back to the rocket's current position
  IF the other side is STATIC geometry AND the library reports the rocket is being
     pushed out of a triangle THEN
    # the body has already been moved out of the surface it hit; walk it back
    reconstruct that triangle, and move `position` back along the rocket's own velocity
    by the penetration distance divided by the cosine between velocity and the triangle
    normal, plus a tenth for margin

  record the contact at `position` with normal `up`
  freeze the body: disable its collision, zero its linear and angular velocity, zero its
    force and torque, switch gravity off for it, and disable the object
```

**Invariants** — **the recorded position is the whole point of this function**, because it is
where the explosion will be centred. Three things conspire against getting it right: the
callback runs after the body has already been integrated into the surface, the solver's
penetration recovery has already begun pushing it back out, and the rocket's own position is
a frame stale. The correction reconstructs the triangle the recovery is pushing against and
walks the position back along the velocity to the surface. Without it, rockets detonate
*inside* walls and the blast is entirely occluded.

**Passable** materials are ignored outright. That is how a rocket flies through a bush and
detonates on the wall behind it, and it is a property of the material table shared with the
weapon system.

The owner exemption compares against the launcher's **root** owner — the creature holding the
launcher, not the launcher itself — so a rocket cannot kill the person who fired it at the
instant of firing.

**Notes** — the callback handles four combinations of which side is which and whether either
side has user data at all, because a contact may be rocket-versus-object or
rocket-versus-static-triangle. A rebuild whose physics library reports contacts with both
participants identified will delete most of this.

## `PlayContact` — the stop

**Contract** — act on a recorded contact, at a safe moment (from the per-frame update, not
from inside the solver). Idempotent once the rocket has already collided.

```text
FUNCTION play_contact()
  IF no contact recorded THEN RETURN
  IF already collided THEN RETURN
  StopEngine(); StopFlying()
  state = collide
  zero the body's velocities, remove its contact callback, put it to sleep
  snap the object's position to the recorded contact point
  clear the recorded contact
```

**Invariants** — the object's position is set to the contact point **after** the body has been
stopped, so nothing subsequently moves it. This is the position the explosion is centred on.

## The motor

### `StartEngine`, `StopEngine`, `UpdateEngine`, `UpdateEnginePh`

**Contract** — the motor burns for a configured number of milliseconds and applies three
impulses per *fixed physics step*.

```text
FUNCTION start_engine()
  REQUIRE the rocket is no longer anyone's child
  IF there is no motor THEN state = flying; RETURN
  state = engine; time_left = work time
  start the engine effect
  register for per-physics-step updates

FUNCTION update_engine()          # per frame
  IF time_left has run out THEN StopEngine(); RETURN
  time_left = time_left - the frame's elapsed milliseconds

FUNCTION update_engine_physics()  # per fixed physics step
  IF the session is replaying a network correction THEN RETURN
  force = impulse * fixed step
  apply (1 + 1) * force along the rocket's own forward axis
  apply force along the REVERSE of the rocket's current velocity, at a point two units
    behind its origin
  apply impulse_up * fixed step straight up
```

**Invariants** — the motor is timed in **frame** time and applied in **physics step** time.
Those are different rates, so the total impulse delivered over a burn depends on the frame
rate. That is a real defect and it is the shipped behaviour; both games' rockets are tuned
against it.

**Notes** — the three impulses together are a crude flight model rather than a thrust. The
forward impulse is doubled by a factor written as `1 + k_back` with `k_back` fixed at one —
the name says a compensation for the second impulse was intended, and the value says it was
never tuned. The second impulse, applied *against* the current velocity at a point behind the
centre, is a stabilizer: it torques the rocket back toward pointing along its own motion, so
a rocket fired while turning straightens out. The constant upward impulse partially cancels
gravity, which is how an unguided rocket flies almost flat over its burn and then drops.

`fixed_step` is the physics world's own timestep, so the impulses are per-step quantities
and the burn is frame-rate dependent only through its timing, not its magnitude per step.

### The forced detonation deadline

**Contract** — a rocket that has hit nothing by a configured time after launch stops itself
where it is, by synthesizing a contact at its own position with its own facing as the normal.

**Notes** — this is measured against **real** time from launch, not flight distance or
in-world time. Without it a rocket fired at the sky flies forever, holding a body, a light,
two effect emitters and a looping sound, and never becoming reusable.

## Effects

### `UpdateParticles`, `StartEngineParticles`, `StartFlyParticles`, `StopFlyParticles`

**Contract** — place the smoke and the looping sound each frame, behind the rocket and moving
with it.

```text
FUNCTION update_particles()
  IF the flying sound is playing THEN move it to the rocket
  IF neither effect exists THEN RETURN
  velocity = the average of the current body velocity and the previous frame's
  build a frame whose forward axis is the rocket's REVERSED forward axis
  place it one unit behind the rocket along that axis
  hand it to both effects with that velocity
```

**Invariants** — the effects are given the rocket's velocity, so the smoke is left in the air
rather than dragged. The velocity is averaged over two frames because the solver's reported
velocity jitters, and jitter in a trail is visible as a stuttering plume.

The fly effect is asserted to be a *looping* effect, with the effect's own name in the
message. A one-shot trail plays once and leaves the rocket bare, which is a data error a
level author can fix.

**Notes** — the one-unit backward offset is marked in the source as a fake. It exists because
the effect's own emitter is authored at its origin and the rocket's model is about that long;
without it the plume comes out of the nose.

Both effects are stopped by being told to auto-remove and then forgotten, rather than being
destroyed. That lets the last puff finish after the rocket is gone.

### `StartLights`, `StopLights`, `UpdateLights`

**Contract** — a shadow-casting trail light at the rocket's position, with a configured colour
and range, following it each frame. Optional per section; a rocket with lights disabled has
none.

**Notes** — whether the light dies with the motor or persists through the unpowered glide is a
flag (`stop_lights_with_engine`, defaulting to "with the engine") set in code and not read
from data.

## Lifecycle

### `OnH_B_Independent`, `OnH_A_Independent`, `OnH_B_Chield`, `OnH_A_Chield`

**Contract** — the four hooks around gaining and losing a parent. Leaving the launcher is what
starts the flight.

```text
FUNCTION before becoming independent(...)
  owner = the ROOT of whatever currently holds us

FUNCTION after becoming independent()
  IF the level is not ready OR this rocket was never launched THEN RETURN
  become visible; StartFlying(); StartEngine()
```

**Invariants** — the owner is captured *before* the detachment, because afterwards there is no
parent to ask. The launched flag distinguishes a rocket that was fired from one that was
merely dropped out of an inventory, which must not ignite.

### `activate_physic_shell`

**Contract** — create the body and launch it with the transform, velocity and angular velocity
the launcher supplied. Requires a parent and no existing body.

**Invariants** — the launch transform, not the rocket's current transform, is what the body is
activated at. The launcher computes it from the muzzle, so the rocket appears at the muzzle
rather than wherever the item happened to be drawn.

Every geometry is marked traced, which tells the collision layer to report the last position
of each — that is what the contact correction reads.

### `OnEvent`

**Contract** — a non-authoritative host, on hearing that this rocket exploded, synthesizes a
contact at its own current position so its local copy stops in roughly the right place.

**Notes** — "roughly" is the honest word: the authoritative position travels in the explosion
event, but this path does not read it, so a client's rocket stops where its own prediction
had it. Since the explosion itself is placed from the event, the visible discrepancy is only
the rocket's own wreck.
