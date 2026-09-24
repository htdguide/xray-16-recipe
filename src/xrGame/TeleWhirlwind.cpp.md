# src/xrGame/TeleWhirlwind.cpp

> The whirlwind anomaly's grip: it drags loose objects along the ground into a funnel, lifts and spins them at the centre, destroys what is fragile enough, and throws the rest back out.

**Needs** — [`TeleWhirlwind.h`](TeleWhirlwind.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`ParticlesPlayer.h`](ParticlesPlayer.h.md) · [`Level.h`](Level.h.md) · [`Hit.h`](Hit.h.md) · [`ai/monsters/telekinesis.h`](ai/monsters/telekinesis.h.md) · [`ai/monsters/telekinetic_object.h`](ai/monsters/telekinetic_object.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/PHImpact.h`](../xrPhysics/PHImpact.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`TeleWhirlwind.h`](TeleWhirlwind.h.md); callers name that, not this file.
**Tier floor** — T2: a per-element force controller stepped against the solver's fixed timestep

## Purpose

There is already a telekinesis mechanism in the game — the controller creature lifts an
object, holds it, and throws it at a target. A whirlwind is the same mechanism with a
different intent: it has no target, it grabs everything rather than one thing, and the
grip itself is supposed to be violent enough to break what it holds.

This file is the whirlwind's specialization of that mechanism, and essentially all of it
is one routine: the force field that pulls an object in. The rest — capture, release,
destruction — is short.

The force field is not a simple attraction, and the three things that make it a *whirlwind*
rather than a point of gravity are each worth naming before the pseudocode:

1. **The funnel.** The target point is not the anomaly's centre. For an object far away
   the target's height is divided down toward the ground, so distant objects are dragged
   along the floor and only lifted as they arrive. A straight pull to the centre would
   make objects arc through the air from across the room, which reads as a tractor beam,
   not a vortex.
2. **The overshoot brake.** A pure attractive force makes anything with sideways velocity
   orbit and escape. Each step predicts where the object will be after one physics step;
   if that prediction is *further* from the target and the object is moving outward, the
   force is replaced by the exact corrective force that redirects the velocity onto the
   line to the target. The result is that things spiral in and cannot slingshot out.
3. **Gravity is cancelled per element**, not disabled. The solver keeps gravity on — the
   object must still fall normally if it leaves — and the field adds an equal and opposite
   vertical force so that the attraction operates in an effectively weightless field. That
   is what makes a heavy crate and a light bucket converge at the same rate, which looks
   supernatural and is the point.

## State

```text
ENUM GripState = raising | keeping | none          # from the shared telekinesis mechanism

RECORD Whirlwind EXTENDS Telekinesis
  centre               : vector          # written each frame by the anomaly that owns this
  keep_radius          : real = 1        # inside this, an object stops being pulled and starts orbiting
  throw_power          : real = 100
  owner                : GameObject      # the anomaly; damage and particles are attributed to it
  pending_impulses     : queue<Impulse>  # launch impulses for the debris of a destroyed object
  destroy_particles    : text

RECORD Impulse
  force : vector      # direction times magnitude; not normalized
  point : vector      # always the body's own origin
```

```text
RECORD WhirlwindObject EXTENDS TelekineticObject
  whirlwind    : Whirlwind
  throw_power  : real
  destroyable  : bool     # decided once at capture, from the object's destructible interface
```

**Invariants**

- An object is captured at most once. A second capture attempt on an already-held object
  is refused, because two grips would sum their forces and launch it.
- While held, the object's air resistance is zero. Drag proportional to speed would fight
  the field's own predictive correction and make the motion mushy and unpredictable.
- The pending-impulse queue is drained by whoever spawns the debris; it exists only
  between a destruction and the spawning of its fragments.

## `activate`

**Contract** — captures an object into the whirlwind, through the shared telekinesis
mechanism, and then hands the newly captured entry this whirlwind's throw power. Returns
the entry, or nothing if the capture was refused.

## `CTeleWhirlwindObject::init`

**Contract** — prepares one captured object. Refuses — by returning failure — if this
object is already held. Otherwise it zeroes the object's air resistance, leaves gravity
enabled on the body, and records once whether the object can be destroyed at all, so the
per-step routine never has to ask again.

## `raise`

**Contract** — called once per physics step for a captured object that is still being
drawn in. Applies a force to every active element of the object's body assembly, then
checks whether the object has arrived and, if so, switches it to the orbiting state.
Does nothing if the object has no active body.

```text
FUNCTION raise()
  body <- object.shell
  IF body IS none OR NOT body.active THEN RETURN
  body.air_resistance <- 0 ; body.gravity <- on

  heaviest <- body.elements[0]
  FOR EACH element IN body.elements
    IF element.mass > heaviest.mass THEN heaviest <- element
    IF NOT element.active THEN CONTINUE

    pos <- element.centre_of_mass

    # --- the funnel: lower the target for distant objects ---
    horizontal <- length of (centre - pos) projected onto the ground plane
    target <- centre
    IF horizontal > 1 THEN target.height <- target.height / horizontal

    offset <- target - pos ; dist <- length(offset)
    accel  <- strength / dist^3                   # steeply local: see note
    IF dist < tiny THEN
      accel <- 0 ; dir <- a random direction      # degenerate: object is exactly on target
    ELSE
      dir <- offset / dist

    # --- one-step prediction ---
    vel          <- element.velocity
    predicted_v  <- vel + dir * (accel * physics_step)
    predicted_p  <- pos + predicted_v * physics_step
    predicted_d  <- distance(target, predicted_p)

    IF predicted_d > dist AND predicted_v points outward AND predicted_v is not negligible THEN
      # --- the overshoot brake: aim the velocity at the target instead of adding pull ---
      motion_dir  <- normalize(predicted_v)
      wanted_vel  <- motion_dir * (offset projected onto motion_dir) / physics_step
      force       <- (wanted_vel - vel) * element.mass / physics_step
    ELSE
      force <- dir * accel * element.mass

    element.apply_force(force + up * (object.effective_gravity * element.mass))

  # --- arrival ---
  IF distance(centre, heaviest.centre_of_mass) < keep_radius AND destroyable THEN
    body.force <- 0 ; body.torque <- 0
    body.velocity <- 0 ; body.angular_velocity <- 0
    switch to the orbiting state
```

**Invariants**

- Arrival is judged by the **heaviest** element, not by the body's origin or its average.
  A jointed object — a chain, a corpse, a hinged sign — has parts that reach the centre
  long before its mass does, and switching state on one of those would leave most of the
  object outside the vortex.
- All velocity and force are zeroed at the transition. The orbiting state imposes its own
  motion and would otherwise inherit whatever momentum the approach ended with.
- An object that cannot be destroyed never enters the orbiting state at all. It is pulled
  in forever and released only when the anomaly releases it — which is what makes
  indestructible things circle the vortex's floor rather than hang spinning at its eye.

**Notes** — the inverse-**cube** falloff is far steeper than physical gravity's inverse
square, and it is what confines the effect: at twice the distance the pull is an eighth,
so the vortex's reach ends sharply instead of tugging at the whole room. A rebuild
substituting inverse-square gets an anomaly that disturbs objects far outside its visible
extent.

The predictive brake computes the corrective force from the object's own mass and the
step duration, which makes it exact for one step regardless of mass — this is an impulse
solved directly rather than a spring, so it does not oscillate and needs no damping
constant.

## `keep`

**Contract** — called once per physics step for an object that has arrived. Holds it at
the eye of the vortex with gravity disabled and spins it. Returns it to the drawing-in
state if it drifts back out.

```text
FUNCTION keep()
  body <- object.shell
  IF body IS none OR NOT body.active THEN RETURN
  body.air_resistance <- 0 ; body.gravity <- off      # weightless at the eye

  heaviest <- the element with the greatest mass
  heaviest.torque <- constant spin about the vertical axis

  IF distance(centre, heaviest.centre_of_mass) > keep_radius * 1.5 THEN
    body.force <- 0 ; body.torque <- 0
    body.velocity <- 0 ; body.angular_velocity <- 0
    body.gravity <- on
    switch to the drawing-in state
```

**Invariants** — the exit radius is **one and a half times** the entry radius, and the
factor is the whole reason the two states do not flicker. An object hovering exactly at
the boundary would otherwise switch state every step, alternating between gravity on and
off, and visibly stutter.

**Notes** — the routine walks every element computing a velocity-damping force and then
discards it without applying it. The intent is plainly to damp residual motion at the eye;
as shipped the only thing holding the object is the fact that its velocity was zeroed on
entry and gravity is off. A rebuild should either apply the damping or drop the
computation. **Could not recover**: nothing — the loop is simply dead.

The spin is a fixed torque on one element, not a rotation of the whole assembly, so a
jointed object flails at the eye rather than turning rigidly. That reads better than a
clean rotation and is probably why it was left that way.

## `release`

**Contract** — lets an object go. Re-enables gravity, then either destroys the object or
throws it, and returns the entry to the released state. Does nothing for an object that
has already been destroyed or has no active body.

```text
FUNCTION release()
  IF object IS gone OR its body is inactive THEN RETURN
  body.gravity <- on

  outward <- object.position - centre ; dist <- length(outward)
  IF dist > 0.2 THEN
    outward <- outward / dist
    impulse <- throw_power / dist^2         # nearer means harder: see note
  ELSE
    outward <- a random direction           # degenerate: no outward to speak of
    impulse <- throw_power * 100

  destroyed <- false
  IF dist < 2 * object.radius THEN destroyed <- destroy_object(outward, throw_power * 100)
  IF NOT destroyed THEN body.apply_impulse(outward, impulse)
  switch to the released state
```

**Invariants**

- The launch impulse grows as the inverse square of the distance from the centre, so an
  object at the eye is fired violently and one at the edge merely nudged. Combined with
  the inverse-cube pull, the whole anomaly is "nothing happens until you are close, then
  everything happens".
- The destruction test uses the object's **own radius**, not a fixed distance: a large
  object counts as "at the centre" from further out than a small one, because it is at the
  centre as soon as any part of it is.
- Degenerate cases are not errors. An object exactly at the centre has no meaningful
  outward direction, so one is chosen at random and the impulse is scaled up rather than
  being left as a division by nearly zero.

## `destroy_object`

**Contract** — attempts to break a captured object. Returns whether it was broken. An
object with no destructible interface cannot be, and is thrown instead.

```text
FUNCTION destroy_object(direction, magnitude) -> bool
  destructible <- object.destructible_interface
  IF destructible IS none THEN RETURN false
  destructible.remove_from_physics_world()
  destructible.destroy(attributed_to: whirlwind.owner)
  IF single player THEN
    queue one launch impulse of (direction, magnitude * 10) per debris piece the
      destructible declares                            # see note
  IF object plays particles THEN
    play the whirlwind's destruction effect at the object's root bone, attributed to
      the whirlwind's owner
  RETURN true
```

**Invariants**

- The object is withdrawn from the physics world *before* it is destroyed, so the solver
  never holds a body whose owner is mid-teardown.
- Destruction is attributed to the anomaly, not to the object: the fragments spawn as the
  anomaly's doing, and anything watching for who destroyed what sees the anomaly.
- The debris impulses are queued only in single player. In multiplayer the fragments are
  spawned by the server and replicated, so the client must not give them its own velocity
  or the two ends diverge.

**Notes** — one impulse is queued per declared debris piece and every one of them is the
same vector; the loop exists to make the queue's length match the fragment count, since
the fragment spawner draws one impulse per fragment. A rebuild should hand the spawner one
impulse and a count.

## `add_impact` / `reserve_impact` / `draw_out_impact` / `clear_impacts`

**Contract** — the pending-impulse queue: append a direction and magnitude as a single
force vector applied at the body's origin, reserve capacity for a known count, remove and
return the oldest, and empty it. Drawing from an empty queue is a programming error and is
asserted rather than handled.

**Notes** — the drawn-out value returns the **force vector**, not a unit direction, along
with its own magnitude — so a caller multiplying the two applies the force squared. A
normalization step exists in the source but is disabled. Whichever convention a rebuild
picks must match its impulse spawner; the shipped pairing is the one that must be
reproduced if the shipped debris behaviour is to be reproduced. **Could not recover**:
which of the two is correct.

## `clear_notrelevant`

**Contract** — drops every held entry whose object has vanished or is being destroyed.
Called when the anomaly's object set may have gone stale. This is the whirlwind's
equivalent of the engine-wide rule that nothing may hold a reference to a departing
entity.

## `can_activate`

**Contract** — whether an object may be captured: anything that exists. A whirlwind grabs
indiscriminately, which is the difference between it and the creature telekinesis it
inherits from, where the choice of target is the whole point.

## `fire` / `raise_update` / `play_destroy` / `clear` / `set_throw_power`

**Contract** — the first three do nothing. A whirlwind has no target to fire at, so both
aiming overloads are deliberately empty and must not fall through to the targeted
telekinesis they inherit. The per-step hook that would release an object stuck too long is
empty as well, so **nothing times out**: an object with no destructible interface, held by
a whirlwind whose owner never releases it, circles indefinitely. A rebuild should decide
whether that is acceptable rather than inherit it by omission.

`clear` and `set_throw_power` are plain delegation and assignment.
