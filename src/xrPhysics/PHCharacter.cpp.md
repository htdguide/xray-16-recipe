# src/xrPhysics/PHCharacter.cpp

> The part of a character controller that is the same for the player and for every
> creature — the single body, the upright constraint, sleep, network state, and the probe
> callback.

**Needs** — [`PHCharacter.h`](PHCharacter.h.md) · [`PHActorCharacter.h`](PHActorCharacter.h.md) · [`PHAICharacter.h`](PHAICharacter.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`Physics.h`](Physics.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHCharacter.h`](PHCharacter.h.md)
**Tier floor** — T1: it writes solver body state directly and steps a body outside the
solver's own loop.

## Purpose

The shared base of both controllers. It is thin — most of the character's behaviour is in
[`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) — and what it holds is the decisions
that are true of *every* character regardless of who drives it.

## construction defaults

**Contract** — a fresh character has no body, no material (the no-material index in both the
last-touched and the injurious slots), the no-restriction size class, is actor-movable, and
has its sleep thresholds overridden to values far tighter than the world defaults:

```text
sleep velocity     0.0001   # world default for a rigid body is 0.001
sleep acceleration 0.001    # world default is 0.1
```

**Notes** — those two overrides are the file's most consequential constants. A character
must be **much** harder to put to sleep than a crate, because a sleeping character does not
respond to the ground moving under it, does not slide down a slope it should slide down, and
— worst — does not wake when the player presses a movement key that has not yet produced a
force. Two orders of magnitude on the acceleration threshold is the margin that makes a
standing character stay simulated.

The creation step is stamped with the no-value marker and filled in on creation; it is used
to reject network state that predates the character.

## the upright constraint

**Contract** — `fix_body_rotation` zeroes the body's angular velocity and resets its
orientation to identity. Called every step by the concrete controllers.

**Notes** — this is the whole of "a character does not fall over". Note it is not a
constraint given to the solver, it is a *post-hoc overwrite*: the solver is allowed to
compute a tumble and then it is thrown away. That is cheaper than an upright joint and
cannot fight with other constraints, but it also means the energy the solver put into
rotation vanishes, so a character hit hard from the side gets less knock-back than momentum
would predict. A rebuild that uses a real upright constraint will find characters get
shoved further.

## sleep and wake

**Contract** — disabling deactivates the object, stops the body, and **clears the
interpolation history**; enabling reactivates and restarts the body, but only if the
character exists.

**Invariants** — clearing the interpolation history on sleep is required, not tidy: a
sleeping character that later wakes somewhere else (teleported, or restored from a save)
must not render a smooth slide from its old position.

Freeze and unfreeze are distinct from sleep: they stop the body without deactivating the
object, and are used by the activation procedures.

## network state

**Contract** — export and import as declared in [`PHCharacter.h`](PHCharacter.h.md).

```text
FUNCTION get_state() -> NetState
  position, previous_position from the interpolation window
  linear_velocity, standing force from the body
  angular velocity = 0, orientation = identity, torque = 0   # the upright constraint
  enabled = whether the OBJECT is active, not whether the body is enabled

FUNCTION set_state(s)
  seed BOTH interpolation slots: previous from s.previous_position, current from s.position
  set position, velocity and force
  enable or disable to match s.enabled
  ASSERT the body's state is finite            # see Notes
```

**Notes** — the exported *enabled* flag reports the physics object's activity, not the
body's. Those can differ for a step, and taking the body's would make a character that is
mid-activation appear asleep to the other end of the connection.

The finiteness assertion on import is the last line of defence named in the
[runtime invariants](../../SYSTEM-REQUIREMENTS.md#6-conformance): a non-finite value
arriving over the network propagates to every body that touches this one within a few steps,
and the world is unrecoverable. A rebuild should validate on import, not assert.

## `CutVelocity`

**Contract** — clamp the body's linear speed to a limit while preserving where the body
would have ended up.

```text
FUNCTION cut_velocity(limit)
  IF |velocity| <= limit          RETURN     # nothing removed, nothing to compensate
  limited   = velocity clamped to limit
  removed   = limited - velocity             # the part being taken away
  set the body's velocity to `removed` and its angular velocity to zero
  integrate the body by exactly one fixed step        # apply the removed motion NOW
  set the body's velocity to `limited`
```

**Notes** — the middle three lines are the decision and they look wrong at first reading.
The body is *moved by the velocity that is being removed*, so that the position it reaches
matches the unclamped trajectory for this one step, while its outgoing velocity is the
clamped one. Without this, every clamp leaves the body short of where the solver placed it
and a sequence of clamps — which is exactly what the activation procedures do — drags the
body backwards. It requires the ability to integrate a single body outside the solver's own
loop, which is the sharpest demand this chapter makes on the
[dynamics seam](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics).

## `GetSavedVelocity`

**Contract** — reports the velocity recorded before the last step when the character is
awake, and the live velocity when it is asleep.

**Notes** — the game layer uses this for animation speed, and it wants the value the
character *had* over the last step rather than the one it ends with, because the end value
includes the current step's contact impulses and spikes on every footfall.

## the movement probe callback

**Contract** — `virtual_move_collide_callback` handles a contact for a body that is testing
a path rather than moving along it.

```text
FUNCTION virtual_move_collide(do_collide, i_am_first, contact, mat_a, mat_b)
  IF already suppressed                       RETURN
  suppress the normal contact                 # we will build our own
  IF the OTHER surface is passable            RETURN      # walk through it
  IF the other shape belongs to the SAME game object   RETURN
  contact.friction = 0                        # slide, never stick
  contact.softness = 0.01                     # slightly compliant, so it doesn't jolt
  build a contact constraint attached to MY body and to NOTHING on the other side
  add it to my object's active island
```

**Invariants** — attaching one side to nothing is what makes this a probe: the world stops
the probing body, and the probing body does not push the world. A one-sided contact is the
mechanism used for this everywhere in the chapter.

**Notes** — zero friction is deliberate. A probe that could stick to a surface would report
a path blocked that the character could actually slide along.

The passable-material test appears in nearly every callback in this chapter: a material
flagged passable — foliage, cloth, decorative rails — generates no contact at all. It is the
single most-used material flag in the physics module.

## the two factories

**Contract** — create an AI controller or an actor controller. The only place in the module
where the concrete types are named.
