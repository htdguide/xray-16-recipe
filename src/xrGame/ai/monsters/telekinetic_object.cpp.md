# src/xrGame/ai/monsters/telekinetic_object.cpp

> One levitated object's life: rise against switched-off gravity, hover in a dead band, then be dropped or thrown.

**Needs** — [`telekinetic_object.h`](telekinetic_object.h.md) · [`telekinesis.h`](telekinesis.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`telekinetic_object.h`](telekinetic_object.h.md)
**Tier floor** — T2: accelerations injected into a physics step, plus timers

## Purpose

This file decides what levitation *looks like*. Three choices produce the whole effect and
a rebuild that gets them wrong will have objects that snap to a height, or that drift away,
or that never settle.

1. **Gravity is switched off and replaced.** The object's own gravity is disabled at
   activation and re-enabled at release or throw. Everything the controller does in between
   is an acceleration it supplies itself.
2. **Hovering is a dead band, not a spring.** There is no error term and no damping. The
   object is pushed hard down if it is above the band, hard up if it is below, and pushed in
   a *random direction* when it is neither. That random push is the reason levitated debris
   wobbles instead of hanging still, and it is the single most recognisable part of the
   effect.
3. **Rising is capped by a clock, not only by height.** An object that cannot reach its
   target height — because it is under an overhang, or too heavy for the force to matter —
   begins hovering anyway after five seconds. Without that cap a poltergeist under a low
   ceiling would hold objects forever.

## State

See [`telekinetic_object.h`](telekinetic_object.h.md). Three timing constants are
hard-coded here and are not configurable from data:

```text
CONSTANT flight_duration      = 3000 milliseconds   # a thrown object is forgotten after this
CONSTANT rise_time_limit      = 5000 milliseconds   # rising gives up and hovers after this
CONSTANT keep_impulse_update  =  200 milliseconds   # dead: see Notes
```

## `init`

**Contract** — refuses any object with no physics body. Otherwise stamps the rising phase,
records the owner, computes the absolute target height as the object's current height plus
the requested lift, stores strength, hold duration and the rotate flag, zeroes the hover
timestamps, and disables the object's gravity. Returns whether it accepted the object.

## `update_state` — the coarse phase machine

**Contract** — one tick of the object's own clock, dispatched on phase. Rising advances to
hovering once the object is high enough *or* the rise time limit has passed, and otherwise
tumbles the object if that was asked for. Hovering releases once the hold duration has
elapsed. Thrown releases once the flight duration has elapsed. Released does nothing.

```text
FUNCTION update_state()
  SELECT phase
    rising:
      IF reached_target_height() OR rise_timed_out()
        begin_hovering()
      ELSE IF rotate
        tumble()
    hovering:
      IF hold_elapsed() THEN release()
    thrown:
      IF flight_elapsed() THEN release()
    released:
      # nothing; the owner will drop this record
```

**Notes** — an earlier design released an object that failed to reach its height rather
than letting it hover; that alternative sits disabled beside the live line. The shipped
behaviour is the forgiving one.

## `raise` — the per-step rising force

**Contract** — applies an upward gravitational acceleration to the object's first physics
element, scaled by the **square of the element count** times the strength. Keeps the hold
sound following the object. Does nothing when the object has lost its physics body, and
nothing on a client that is not the authority.

```text
FUNCTION raise(step)
  IF object has no active physics body THEN RETURN
  element_count = object.physics.element_count
  lift = up * (element_count * element_count * strength)
  IF this machine is the authority
    object.physics.first_element.apply_gravity_acceleration(lift)
  update_hold_sound()
```

**Invariants** — the force is applied to **one** element of a multi-element body, and is
scaled by the element count squared to compensate. That is why a ragdoll or a jointed prop
rises at roughly the same rate as a single-piece crate rather than dragging by one limb. It
is a calibration, not a physical law, and a rebuild whose physics seam lets it apply the
acceleration to every element should do that instead and drop the squaring.

**Notes** — the function takes the physics step length, multiplies it into the strength,
and then never uses the result: the lift is computed from strength alone. Because the
quantity injected is an *acceleration* and not an impulse, the outcome is step-rate
independent anyway, so the dead line changes nothing — but a rebuilder who "fixes" it by
multiplying the lift by the step will make levitation depend on the physics rate.

## `keep` — the per-step hovering force

**Contract** — the dead band. Compares the object's height against the target height offset
by 60 centimetres and pushes down, up, or in a random direction; the push is always the same
magnitude, five units of acceleration, applied to the first element. Keeps the hold sound
following the object.

```text
FUNCTION keep()
  IF object has no active physics body THEN RETURN
  IF object.position.y > target_height + 0.6
    direction = down
  ELSE IF object.position.y < target_height + 0.6
    direction = up
  ELSE
    direction = normalize(random vector in the unit cube)   # the wobble
  IF this machine is the authority
    object.physics.first_element.apply_gravity_acceleration(direction * 5)
```

**Invariants** — the two comparisons are strict and against the same value, so the random
branch is reached only on exact equality — effectively never in floating point. The wobble
is therefore not what makes debris shake; the alternation between the up and down pushes
is, and it oscillates around a point 60 centimetres *above* the nominal target height. A
rebuild that implements a real dead band with a width will get a visibly calmer effect, and
a rebuild that implements a spring will get a calmer one still. The shipped look is the
bang-bang controller.

**Notes** — the hover force is *not* scaled by element count, unlike the rising force, so
multi-element props hover less firmly than they rise. There is a disabled rate limiter here
too — the 200 millisecond constant — which would have applied the hover force at most five
times a second; it is not compiled, and the timestamp it maintained is written every call
and read nowhere.

## `release`

**Contract** — re-enables the object's gravity and applies a small downward impulse of half
the object's mass, so that a body the physics library had put to sleep actually starts
falling instead of hanging. Switches the phase to released. Does nothing when the physics
body is gone.

## `fire` — throw with a power factor

**Contract** — stamps the thrown phase, re-enables gravity, and applies an impulse along the
direction from the object to the target to **every** element, each getting the total impulse
divided by the element count. The total is twenty times the mass times the caller's power
factor. Aim is a straight line to the target with no ballistic compensation, so this form
undershoots at distance; it is the close-range form.

## `fire_t` — throw with a flight time

**Contract** — stamps the thrown phase, re-enables gravity, and *solves* for the launch
velocity that puts the object at the target after the given flight time under the object's
effective gravity, then assigns that velocity directly rather than applying an impulse.
Plays the throw sound at the object and stops the hold loop. This is the form used when the
throw must actually connect.

**Invariants** — assigning a velocity rather than applying an impulse makes the throw
independent of mass, which is what makes the arc predictable. The two throw forms therefore
behave differently in kind, not only in degree, and callers choose deliberately.

## `check_height` / `check_raise_time_out` / `time_keep_elapsed` / `time_fire_elapsed`

**Contract** — the four predicates. Height is a plain comparison against the absolute target
and reports *true* when the object is gone, so a destroyed object stops rising rather than
hanging in the rising phase forever. The three timers compare their phase's start stamp
against the clock.

## `rotate`

**Contract** — applies an impulse in a uniformly random direction with magnitude two and a
half times the object's mass. Called once per coarse tick while rising, which is what makes
levitated debris tumble as it goes up rather than sliding up flat.

## `enable` / `can_activate` / `update_hold_sound`

**Contract** — `enable` wakes the physics body, called each step by the controller so a
held object is never put to sleep. `can_activate` is the acceptance test: an object with a
physics body. `update_hold_sound` starts the hold loop at the object's position if it is not
playing and moves it if it is, which is how the sound follows the object without the caller
tracking it.
