# src/xrPhysics/ActorCameraCollision.cpp

> Keeps the first-person camera's near plane out of walls by solving a tiny
> one-body physics problem on demand, outside the world's own step.

**Needs** — [`ActorCameraCollision.h`](ActorCameraCollision.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`GeometryBits.h`](GeometryBits.h.md) · [`Geometry.h`](Geometry.h.md) · [`matrix_utils.h`](matrix_utils.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`ActorCameraCollision.h`](ActorCameraCollision.h.md)
**Tier floor** — T1: it runs the solver re-entrantly on one body between world steps and
writes contact constraints by hand.

## Purpose

A first-person camera sits roughly where the player's head is, but the near plane sticks
out in front of it. Walk into a wall and the near plane crosses it, and the player sees
through the world. The fix chosen here is not a ray cast and not a spring arm: the camera's
near-plane frustum is given a **physical body** and pushed out of the wall by the same
solver that moves everything else, then the resolved position is read back.

That is an expensive answer to a cheap-looking problem, and the reason it was chosen is
concavity. A ray or a sphere sweep has to pick one direction to retreat along; a small box
settled by a constraint solver finds a position satisfying *every* touching surface at once,
which is what a corner, a doorway or a crouch under a beam demands. The price is that the
world must be stepped, which is why everything in this file is written around one
constraint: **the world is frozen for the duration**, so the resolve cannot advance the
game's own simulation.

## State

```text
RECORD CameraCollisionModule           # one instance, module-scope
  shell         : optional<Shell>      # the camera's body; built lazily, reused forever
  collided      : bool                 # latch set by the contact callbacks this pass
  stepping      : bool                 # are we in the push-out phase, or only probing?

CONSTANTS
  skin_depth              = 0.04       # a contact shallower than half this is not a hit
  character_skin_depth    = 0.4        # the extra clearance kept from other characters
  character_shift_z       = 0.3        # how far forward the anti-character cylinder sits
  character_shift_y       = 0.8        # how far below the camera that cylinder hangs
  soft_cfm_geometry       = 0.01       # compliance against the level
  soft_cfm_characters     = 0.05       # compliance against another creature: softer
  correction_steps        = 100        # give-up bound on the push-out loop
  camera_mass             = 10         # see Notes
  camera_inertia_radius   = 10_000_000 # see Notes
```

**Invariants** — `collided` and `stepping` are read by the contact callbacks, which the
solver invokes from inside a step; they are therefore only meaningful while exactly one
camera resolve is in flight. The whole file assumes a single camera and asserts that the
world is not already processing when it is entered.

## the camera shell

**Contract** — build once, on first use, and rebuild only when the owning entity changes
(the player switches bodies, or a new level's actor appears). It is a single-element shell
carrying two shapes:

- a **box**, sized each frame to the camera's near plane plus the skin depth, which
  collides with the level and with physical objects;
- a **cylinder**, standing in front of and below the camera, which collides *only* with
  other characters.

Both are attached to the same body. Gravity is off, collision is left disabled between
uses, and the body is asleep except during a resolve.

**Invariants** — the cylinder is marked to ignore static geometry. Its only job is to keep
the camera from entering another creature's volume; letting it feel walls too would fight
the box and stall the push-out loop.

**Notes** — the mass is set from a sphere of ten-million-unit radius and then rescaled to
ten units of mass. The effect is a body with enormous rotational inertia and modest linear
inertia: it translates freely under contact and is effectively impossible to spin. That is
the cheapest way to say "this body may slide out of a wall but must never tumble", and it
matters because the camera's orientation is dictated by the player's look input and is
overwritten every iteration anyway.

The two shapes deliberately get different contact softness. Against level geometry the
camera is nearly rigid (it must not be allowed to sink into a wall); against another
character it is five times more compliant, so that brushing past a creature nudges the view
rather than snapping it.

## contact handling

**Contract** — three callbacks share one body of rules.

```text
FUNCTION camera_contact(contact, i_am_first, material_a, material_b)
  suppress the solver's own contact                  # we decide everything ourselves
  IF the opposite material is passable       RETURN  # foliage, cloth: see through it
  IF the opposite shape belongs to MY game object    RETURN
  IF the opposite object declines camera collision   RETURN   # a per-object opt-out
  IF contact.depth > skin_depth / 2
      collided = true                                # the probe's whole answer
  IF NOT stepping                            RETURN  # probing only; build no constraint
  contact.friction = 0                               # slide out, never stick
  build a contact constraint and attach it to MY body and to NOTHING on the other side
  add it to my object's active island
```

**Invariants** — the one-sided attachment is the rule the whole file rests on: the world
stops the camera, the camera never pushes the world. A camera body with a ten-unit mass
that could push would shove crates around as the player looked at them.

**Notes** — the half-skin-depth threshold is the difference between "touching" and "inside".
The box is inflated by a full skin depth before the query precisely so that a camera resting
*against* a wall still reports clear; only genuine penetration past half that margin
counts.

The anti-character callback inverts the default: it suppresses every contact and re-enables
only against another *stalker*-class creature. Monsters are excluded, because their
collision volumes are authored loosely and the camera would be shoved by a rat.

The per-object opt-out (an object declaring it does not collide with the actor camera) is
the data-driven escape hatch for props the designers wanted the camera to pass through —
a mod-added addition, and the only place in this file where the game layer gets a vote.

## sizing the box from the camera

**Contract** — the box is the near plane made solid: half-width and half-height from the
camera's field of view at the near distance, depth equal to half the near distance, with
the transform built from the camera's right/up/forward basis and its origin pushed half a
near-distance forward so the box straddles the plane. Every dimension is then grown by the
skin depth.

**Notes** — when the box is installed on the body it is further stretched — twice as deep,
half again as tall — and shifted back and down by half of each. The camera therefore
defends a volume somewhat larger than its own near plane, biased *behind and below* the
plane. That bias is what stops the wall appearing between the player's eye and the near
plane when the player looks down while walking into it; the numbers are authored by eye and
are not derivable.

## `collide_camera`

**Contract** — takes a camera and the owning entity, and moves the camera's position so its
near plane is clear. It blocks (it steps the solver), allocates nothing per call after the
first, and must not be called while the world is stepping.

```text
FUNCTION collide_camera(camera, near_distance, actor)
  ensure the camera shell exists and belongs to `actor`
  box, transform = box from the camera
  IF box and transform are unchanged since last frame    RETURN   # nothing moved
  install the box and the anti-character cylinder, place the body at `transform`
  resolve_and_move(transform, actor)
  camera.position = body position, pulled BACK along the view direction by half the near
                    distance                            # undo the forward offset
```

```text
FUNCTION resolve_and_move(transform, actor)
  collided = false ; stepping = false
  disable the actor's OWN movement collision      # see Invariants
  enable the shell's collision and run one collision pass with no solve
  IF NOT collided
      restore and RETURN                          # the common case: one query, no step
  stepping = true
  FOR i IN 1 .. 100
      zero the body's linear and angular velocity
      reset the body's orientation to the camera's           # each iteration, not once
      collided = false
      step this body alone
      IF NOT collided   BREAK
  stepping = false
  restore the actor's movement collision, disable and sleep the shell
```

**Invariants** — the actor's own character collision is switched off across the resolve.
Without that, the camera body — which lives inside the player's own capsule — is pushed out
of the player before it is pushed out of the wall, and the view ends up somewhere behind the
character's shoulders.

Velocity is zeroed and the orientation restored at the *top of every iteration*, not once
before the loop. This makes the loop a sequence of independent single-step relaxations
rather than a simulation: the body has no momentum to carry into the next iteration, so it
cannot overshoot or oscillate, and the hundred-iteration bound is a give-up, not a budget.

**Notes** — the early-out on an unchanged box and transform is the reason this is affordable
at all. A standing player pays one comparison per frame. The comparison is a combined
translation-and-rotation nearness test against the previous pose, so an imperceptible drift
also counts as unchanged.

## `test_camera_box` / `test_camera_collide`

**Contract** — the read-only forms. They install the requested box, run a single collision
pass with no solve, restore the box and transform that were there before, and return whether
anything was hit. The camera is never moved and the world is never stepped.

**Invariants** — the previous box size and transform are saved and put back, because the
persistent shell is shared with `collide_camera`; a probe that left the box resized would
corrupt the next frame's early-out comparison.

**Notes** — `test_camera_collide` differs only in building the box from a camera and then
applying a caller-supplied scale and forward offset. The game layer uses it to ask questions
like "is there room to raise the weapon here" — the same volume test, at a different place
than the camera actually is.
