# src/xrPhysics/PHActivationShape.cpp

> The settle procedure — put a box where something is about to be born, let the
> world push it out, and report where it stopped.

**Needs** — [`PHActivationShape.h`](PHActivationShape.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHObject.h`](PHObject.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`SpaceUtils.h`](SpaceUtils.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`PHValideValues.h`](PHValideValues.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHActivationShape.h`](PHActivationShape.h.md)
**Tier floor** — T1: it freezes and steps the whole world re-entrantly and integrates a
single body by hand.

## Purpose

The engine repeatedly needs to answer "where can this volume actually go?" — a corpse about
to become a ragdoll, an object about to switch from animated to simulated, an explosion
about to claim a sphere of space. The answer is produced by *running physics on a stand-in*:
a box is created at the requested place, and the world is stepped with everything else
frozen until the box is no longer penetrating anything. Its final position is the answer.

This is the one procedure in the chapter that inverts the usual control flow. Normally the
world steps and objects react; here an object steps the world, many times, inside a single
game frame.

## State

```text
RECORD ActivationShape
  body       : Body                 # one body, no shell
  shape      : Shape                # box, cylinder or sphere
  flags      : set of {fixed_rotation, fixed_position, static_environment, gravity}
  safe_state : SafeFixedRotation    # last known-good pose, for divergence recovery

MODULE STATE                        # read by the contact callbacks
  max_depth  : real                 # deepest (or total) penetration seen this iteration

CONSTANTS
  contact_softness   = 1e-10        # very nearly rigid
  contact_error_reduction = 1       # correct penetration fully in one step
  iterations_per_step = 15
  resolve_depth       = 0.01        # converged when the worst penetration is under this
  attempts_per_step   = 10
  inertia_radius      = 100000      # see Notes
```

**Invariants** — the module-scope penetration accumulator means only one activation may be
in flight at a time. Every shipped caller creates the shape on its own stack and destroys it
before returning, which enforces that by construction; a rebuild that wants concurrent
activations must move the accumulator into the record.

The destructor asserts that both the body and the shape are already gone — `Destroy` is not
optional, because an activation shape left in the world is an invisible obstacle.

## `Create`

**Contract** — builds the body at the requested position with the requested shape, registers
it for spatial queries and with an island, wakes it, and installs the depth-measuring
contact callback. The requested position and size are validated first and a bad one is a
hard failure, not a clamp.

**Notes** — the mass is a sphere of hundred-thousand-unit radius rescaled to unit mass:
enormous rotational inertia, ordinary linear inertia. As with the camera shell, this is how
"translate freely, never spin" is said to a solver that has no such switch. Fixed rotation
is also set as a flag by default, so the inertia trick is belt and braces.

## `Activate`

**Contract** — the settle. Takes the target size, the number of growth steps, and per-step
caps on displacement and rotation. Returns true when the box reached its target size with
every penetration under the resolve threshold; false when it ran out of attempts. Blocks for
as long as the loop takes. The world is frozen on entry and unfrozen on exit unless the
caller asks to unfreeze later.

```text
FUNCTION activate(target_size, steps, max_displacement, max_rotation) -> bool
  activate this object ; freeze the entire world ; unfreeze only me

  # --- phase 1: measure how bad the starting position is ---
  install the SUMMING depth callback
  run one collision-only pass                    # touch, do not solve
  max_depth now holds the TOTAL penetration at the spawn point

  # --- derive the speed caps from that measurement ---
  velocity_cap = max_depth / 15 / steps / timestep
  size_cap     = largest target dimension / 15 / steps / timestep
  velocity_cap = min(velocity_cap, size_cap, world's default linear cap)
  angular_cap  = min(max_rotation / 15 / steps / timestep, world's default angular cap)

  install the MAX-depth callback (plus the static-environment callback if asked)
  save every body's velocity state in the world

  # --- phase 2: grow and settle ---
  size_increment = (target_size - current_size) / steps
  FOR each growth step
      grow the shape by size_increment
      attempts = 10
      REPEAT
          converged = false
          FOR i IN 1 .. 15
              max_depth = 0
              step the world
              clamp every body's velocity to (velocity_cap, angular_cap)
              IF max_depth < 0.01 THEN converged = true ; BREAK
      UNTIL converged OR attempts exhausted

  restore every body's saved velocity state
  unfreeze the world
  RETURN converged
```

**Invariants** — the velocity cap is *derived from the measured penetration*, not fixed.
A box spawned one centimetre inside a wall is allowed to move about a centimetre per
iteration; a box spawned two metres inside rock is allowed to move about two metres. That
is what makes the same fifteen iterations sufficient for both cases without letting the
shallow case jitter.

Velocity state for the whole world is saved before the loop and restored after. This is the
load-bearing correctness property of the whole procedure: the world was stepped dozens of
times, and every frozen body nonetheless accumulated impulses from touching the growing box.
Restoring linear velocity, angular velocity and the enabled flag — but *not* position —
means the other bodies keep the shoves they received while forgetting the speed those shoves
gave them. A rebuild that skips this leaves every object near a newly spawned ragdoll
quietly moving.

**Notes** — the two-callback arrangement is easy to misread. The first pass uses a callback
that **sums** penetration depth over all contacts, because the question is "how embedded is
this, overall"; the settle loop uses one that keeps the **maximum**, because the question
becomes "is the worst remaining penetration acceptable". Same field, two different meanings,
and the switch between them is what lets one threshold serve both.

The ten-attempt retry around the fifteen-iteration loop covers the case where growing the
box re-embeds it after the previous size had converged. There is no back-off between
attempts; failing all ten simply reports non-convergence to the caller, who decides what to
do (see [`IActivationShape.cpp`](IActivationShape.cpp.md)).

Contacts during a settle are made nearly rigid — negligible compliance, full error
correction in one step — and their friction is scaled by a module factor that ships set to
zero. Frictionless, rigid contacts make the box slide out along the shortest path instead of
gripping the surface it is embedded in. This is the opposite of what is wanted for gameplay
contacts and is correct only because nothing here is meant to look like motion.

## the static-environment option

**Contract** — when the flag is set, an additional callback turns every contact with static
geometry into a one-sided constraint attached to the settling body and to nothing, and
suppresses the solver's own contact.

**Notes** — the difference from the default path is which side is allowed to move. The
default lets the settling box and a *dynamic* obstacle negotiate; this makes the level
immovable in a way the solver cannot compromise on, for callers that must not be pushed
partway into a wall by a crate on the other side.

## `CutVelocity`

**Contract** — the world's per-step velocity clamp, as this object implements it: when the
linear speed exceeds the limit, the body is set to the *removed* portion of its velocity,
integrated by exactly one timestep, and only then set to the clamped velocity. Angular
velocity is zeroed outright.

**Notes** — the same construction as the character controller's clamp (see
[`PHCharacter.cpp`](PHCharacter.cpp.md)) and for the same reason: the clamp must change the
speed without losing the distance the solver already decided the body should travel this
step. Without it the settle loop would creep backwards against itself and never converge.

## the remaining overrides

**Contract** — the simulated-object duties this type must answer but has nothing to say
about: per-step bookkeeping only records the safe pose; contact tuning is a no-op; the
spatial-query parameters are derived from the shape's bounds; the element count is zero and
there is no network state, because an activation shape never outlives the call that made it
and is never replicated.
