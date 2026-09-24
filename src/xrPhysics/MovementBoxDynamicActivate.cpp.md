# src/xrPhysics/MovementBoxDynamicActivate.cpp

> Grows a character's collision box to a new size while the world is frozen,
> pushing the character out of whatever the larger box now touches.

**Needs** — [`MovementBoxDynamicActivate.h`](MovementBoxDynamicActivate.h.md) · [`PHCharacter.h`](PHCharacter.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`ph_valid_ode.h`](ph_valid_ode.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`MovementBoxDynamicActivate.h`](MovementBoxDynamicActivate.h.md)
**Tier floor** — T1: it steps the world re-entrantly, rewrites body state between steps and
reads per-contact solver feedback.

## Purpose

Standing up in a low tunnel, crawling out from under a truck, or being spawned into the
world at all: each is a change of the character's collision box, and each can put the new
box inside geometry. This file performs the change as a physical process instead of an
assignment — the box is grown in fractions while the world is stepped and the character is
allowed to be squeezed out — and reports whether the character ended up somewhere legal.

It is the character-shaped sibling of [`PHActivationShape.cpp`](PHActivationShape.cpp.md),
and the differences are all consequences of the subject being a character: the body already
exists and is driven by intent rather than forces, so the controller's own tick must be run
between steps; the box is not a free volume but one of a small set of authored shapes; and
the feet need different contact rules from the torso.

## State

```text
MODULE STATE
  saved_callback : ContactCallback       # the character's own callback, parked for the pass
  max_depth      : real                  # worst penetration seen this iteration

CONTACT TUNING, body vs. feet
                             body      feet
  friction_scale             0.0       0.3       # feet may grip; the torso must slide
  depth_to_push_with_force   0.3       0.3
  push_force_scale           10.0      10.0      # multiples of gravity
  contact_softness           tiny      tiny      # world softness scaled by 1e-5
  contact_error_reduction    1.0       1.0
  depth_ignored              0.0       0.05      # feet forgive 5 cm before complaining
  depth_clamp                0.2       0.2       # never report a contact deeper than this
```

**Invariants** — the module-scope penetration accumulator and parked callback mean one box
activation at a time, globally. The pass must restore the character's own contact callback,
its initial-contact handling and its static-contact callback before returning, or the
character is left permanently in probe mode.

## the depth-measuring contact rule

**Contract** — during a box activation the character's contacts are rewritten. The rule is
shared between the body and the feet and differs only in the numbers above.

```text
FUNCTION activation_contact(contact, materials)
  run the character's ORIGINAL callback first        # so game-side effects still happen
  IF either material is passable                RETURN
  effective_depth = contact.depth - depth_ignored
  max_depth = max(max_depth, effective_depth)
  contact.friction = contact.friction * friction_scale

  IF effective_depth > 0.3
      # too deep to solve as a constraint: push instead
      force = 10 * gravity
      push both bodies apart along the contact normal
      wake both objects' shells
      suppress the contact entirely
  ELSE
      make the contact very stiff (tiny compliance, full error reduction)

  clamp contact.depth to 0.2                          # see Notes
```

**Notes** — the two regimes are the decision. A constraint solver asked to remove thirty
centimetres of penetration in one step produces an impulse that launches the body; below
that threshold a stiff contact is well behaved, above it an explicit force is applied
instead and the constraint is dropped. The force is expressed as a multiple of gravity so
the behaviour does not change with the character's mass.

Clamping the *reported* depth to twenty centimetres is a separate protection from the same
family: even in the stiff regime, the solver is never told the full truth about a deep
penetration, so its correction impulse stays bounded. The measured depth used for
convergence is taken before the clamp, so the loop still knows it has not finished.

The feet forgive the first five centimetres of penetration and keep thirty percent of their
friction. Feet are *supposed* to be in the ground — the capsule's lower cap always overlaps
the floor slightly — so counting that as failure would make every activation fail. The torso
gets no such allowance and no friction at all, because a torso that grips a wall while being
squeezed out cannot slide along it.

## the velocity limiter

**Contract** — a per-step object attached to the character's body for the duration of the
pass. It caps horizontal speed and vertical speed separately, and when it caps either, it
*repositions* the body to where the pre-cap velocity would have taken it over one timestep.
It also detects a non-finite velocity or position and restores the last known-good pose.

```text
FUNCTION limiter_after_step()
  v = body velocity
  IF v is not finite
      restore the saved velocity ; capped = true
  ELSE
      horizontal = length of v in the ground plane
      IF horizontal > horizontal_limit   scale both ground components down ; capped = true
      IF |v.vertical| > vertical_limit   clamp the vertical component     ; capped = true
      write v back
  IF capped
      body position = saved position + PRE-CAP velocity * timestep
  IF the body position is still not finite
      body position = saved position - saved velocity * timestep     # step backwards
  save this position and this velocity
```

**Invariants** — horizontal and vertical are limited independently. A character being pushed
out of a ceiling moves almost entirely vertically; one being squeezed out of a doorway moves
almost entirely horizontally. A single speed cap would either starve the first case or let
the second slide across the room.

**Notes** — the "position from pre-cap velocity" rule is the same compensation the character
controller's own clamp performs, restated for a body under an external limiter: the clamp
must not cost the body the distance the solver already gave it. The non-finite fallback —
stepping *backwards* along the last good velocity — is the module's last resort; it prefers
a body one step in the past to a body at infinity, because the latter poisons every object
it touches. See the runtime invariants in
[conformance](../../SYSTEM-REQUIREMENTS.md#6-conformance).

## the contact-force reader

**Contract** — a per-step object that attaches feedback to every contact constraint on a
body before the solve, and after the solve reports the largest force and torque the body
received, split into what acted on *it* and what it inflicted on *others*, with the
self-force further split into vertical and horizontal components.

**Notes** — nothing in the activation path consults these numbers; the class is here as the
measurement instrument the same body of code uses for impact damage and for deciding whether
a squeeze is survivable. The others' torque is divided by the lever arm from the contact
point to the other body's centre, which converts it into a comparable force rather than a
torque — the caller wants to know how hard the character shoved something, not how much it
spun it.

## `ActivateBoxDynamic`

**Contract** — switch the character to candidate box *id* and report convergence. Blocks for
the duration; freezes the world, thaws only this character, and restores everything on exit.

```text
FUNCTION activate_box_dynamic(controller, character_exists, id, iterations, steps, resolve_depth)
  activate the character as a simulated object
  freeze the world ; unfreeze this character only
  park the character's contact callback; install the depth-measuring ones
      (the body rule on the character, the foot rule on the feet)

  IF NOT character_exists
      iterations = 20 ; steps = 1 ; resolve_depth = 0.1     # see Notes

  # how far the box has to change, and therefore how fast the body may move
  travel = character_exists ? |radius(current box) - radius(box[id])|
                            : radius(box[id])
  velocity_cap = travel / 2 / iterations / steps / timestep
  angular_cap  = 22.5 deg / iterations / steps / timestep

  zero the character's force and velocity
  run one controller tick with zero input                  # resync the controller
  attach a velocity limiter at the cap, then multiply its cap by
      (iterations * steps / 5)                             # loose during settling
  switch the character out of initial-contact handling and drop its static callback

  # --- phase 1: settle in the CURRENT box, up to 30 iterations ---
  REPEAT 30 TIMES
      run a controller tick with zero input ; wake the character
      apply an upward force of gravity * mass              # cancel gravity: see Notes
      max_depth = 0 ; step the world
      IF max_depth < resolve_depth   BREAK
      clamp every body's velocity

  restore the limiter's cap to the tight value

  # --- phase 2: grow the box in `steps` fractions ---
  FOR m IN 1 .. steps
      interpolate the box toward candidate `id` at fraction m/steps
      converged = false
      FOR i IN 1 .. iterations
          max_depth = 0
          run a controller tick with zero input ; wake the character
          apply an upward force of gravity * mass
          step the world ; clamp every body's velocity
          IF max_depth < resolve_depth   converged = true ; BREAK
      IF NOT converged   BREAK                              # give up at this size

  restore initial-contact handling, the static contact callback and the parked callback
  detach the limiter ; unfreeze the world
  RETURN converged
```

**Invariants** — the character is held against gravity for every iteration by an explicit
upward force equal to its weight. Without it the character falls while the box grows and the
settle converges on a position below the floor; with it, the only thing moving the character
is contact with what the growing box has hit.

Phase 1 exists because the character may already be penetrating something *before* the
resize is asked for. Settling in the old box first means phase 2 starts from a legal
position, so a failure in phase 2 can be honestly attributed to the new box being too big
for the space.

**Notes** — the three parameter overrides when the character does not yet exist are the
spawn case: there is no previous box, so the whole target radius must be traversed (not a
difference of radii), there is nothing to grow *from* so one step suffices, and the
tolerance is loosened tenfold because a freshly spawned creature at an authored spawn point
is often lightly embedded in the ground by design and insisting on a centimetre would fail
every spawn.

The velocity limiter is deliberately run loose during phase 1 and tight during phase 2 — by
a factor of `iterations * steps / 5`. Phase 1 wants the character to travel however far it
must to escape its current predicament; phase 2 wants it to creep, because every millimetre
of phase-2 motion is a change the player will see. The divisor 5 has no derivation in the
source.

Initial-contact handling and the static-contact callback are switched off for the pass. The
first is the character's own per-contact tuning (slope limits, step climbing), which would
fight the activation rules; the second is the callback that leaves scorch and footfall marks
on surfaces, which would stamp several dozen marks under a character that merely stood up.
