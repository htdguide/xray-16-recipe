# src/xrGame/ClimableObject.cpp

> A ladder: an oriented box placed in the level that publishes the geometric questions a climbing character needs answered — where is the axis, which way is up it, am I in front of it, how far to the top — and that lets a character walk through it from behind.

**Needs** — [`ClimableObject.h`](ClimableObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`xrPhysics/IClimableObject.h`](../xrPhysics/IClimableObject.h.md) · [`xrPhysics/IPHStaticGeomShell.h`](../xrPhysics/IPHStaticGeomShell.h.md) · [`xrPhysics/PHCharacter.h`](../xrPhysics/PHCharacter.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: vector geometry against a static collision volume

## Purpose

A ladder is authored as nothing more than an oriented box with a surface material. This
file turns that box into a *frame*: three named directions — the axis you climb along, the
side you may slide along, and the normal pointing out of the climbing face — and a
collection of measurements taken between a character and that frame. The character's
movement code owns the climbing behaviour; this object owns the geometry, and the split is
what lets one climb routine serve ladders of any size and orientation.

Three decisions carry the file.

**The frame is derived, not authored.** The level editor gives a box with three half-extents
and an orientation. Which of the three axes is the climbing direction is decided at spawn
by *which is the longest* — a ladder is a tall thin box, so its long axis is the one you
climb. The thinnest is the normal. This means an author never labels a ladder's direction
and cannot get it wrong.

**A thin ladder is fattened.** A box thinner than a fixed minimum along its normal is
widened to that minimum and shifted so that the widening grows outward, away from the wall
it is mounted on. Without this the "am I in front of the ladder" test has no tolerance and
a character standing flush against a flat ladder reads as being inside it.

**The ladder collides selectively.** Its collision volume is registered with the physics
world, but the contact callback rejects every contact from a character that is *not*
standing in front of it. So a ladder is solid to someone climbing it and transparent to
someone walking past behind — which is what lets a ladder be modelled as a slab
intersecting the wall it is fixed to.

## State

```text
RECORD Ladder
  axis     : vector   # half-length along the climbing direction, world space, always +Y-ish
  side     : vector   # half-length across the face, world space
  norm     : vector   # half-thickness out of the climbing face, world space
  box      : oriented box     # the collision volume, after the thin-ladder widening
  radius   : real             # the largest half-extent; the object's cull radius
  material : int              # surface material index, for footstep sounds
  shell    : static geometry shell   # the collision registration
```

Invariants that matter:

- the three vectors are *not* unit — each is scaled to the corresponding half-extent, so
  their magnitudes carry the ladder's size and callers that want a direction normalize
  explicitly. Several of the measurements below depend on this.
- `axis` always points up: if the derived axis points downward it is inverted, and `side`
  is inverted with it to keep the frame's handedness. Nothing else in the frame is
  corrected, so the normal may point either way relative to the other two.
- the ladder is excluded from AI vision and does not register with the update scheduler.
  It is inert furniture that answers questions.

## `net_Spawn`

**Contract** — builds the frame from the authored server record and registers the collision
volume. The order is load-bearing: the box half-extents are read and the cull radius
derived *before* the base spawn, because the base's spatial registration needs the radius;
the frame directions are derived *after*, because they are world-space and need the
object's transform, which the base sets.

```text
FUNCTION spawn(record)
  material = material index named by the record
  half     = the record's first shape, read as three half-extents
  radius   = max of the three half-extents
  base.spawn(record)
  remove this object from the AI-visible set    # a ladder is not a thing to be seen

  # pick the frame by comparing the three half-extents; the longest is the climbing axis,
  # the shortest the normal, the middle one the side
  order the three axes by half-extent, descending
  axis = object's longest-axis direction, scaled by its half-extent
  side = object's middle-axis direction, scaled by its half-extent
  norm = object's shortest-axis direction

  IF the shortest half-extent < minimum_width THEN
    widen the box along that axis to minimum_width
    shift = that axis
  norm = norm scaled by the (possibly widened) half-extent

  move the object back by minimum_width along shift, in world space
  # so the widening extends outward from the mounting surface, not into it

  box is re-expressed in the object's own space
  shell = register a static collision volume for this box, with our contact filter

  IF axis points downward THEN
    invert axis and side

  stop taking per-frame processing     # nothing to update
```

**Notes** — the "order the three axes" step is written in the original as a three-way
compare macro whose branches are textually near-identical. The decision it encodes is the
one stated above, and a rebuild should write it as a sort.

## `net_Destroy`

**Contract** — releases the static collision volume after the base tears the object down.
This is the conformance invariant: an object removed from the world must not remain in the
physics world.

## `Axis` / `Side` / `Norm` and `DDAxis` / `DDSide` / `DDNorm`

**Contract** — the frame, in two forms. The plain accessors yield the scaled vectors,
magnitude and all. The `DD` forms normalize into a caller-supplied vector and return the
magnitude that was removed, so one call yields both direction and extent. That pairing —
"decompose into magnitude and direction in one step" — is the idiom the whole file is
written in, and it is why so many of these functions return a float and write through a
parameter.

## `LowerPoint` / `UpperPoint`

**Contract** — the two ends of the climb, each offset *outward along the normal* by the
ladder's half-thickness. So the mount and dismount points sit on the climbing face, not on
the box's centre line, which is where a character's feet actually need to be.

## `POnAxis` / `DToAxis` / `DDToAxis`

**Contract** — project a character's foot centre onto the ladder's axis line, and give the
vector from the character to that projection. `DDToAxis` returns its magnitude and leaves
a unit direction behind. This is the "how do I get onto the ladder" measurement.

## `DSideToAxis` / `DDSideToAxis`

**Contract** — the same offset, resolved onto the side direction alone: how far the
character is off-centre across the ladder's face. `DDSideToAxis` returns the *unsigned*
distance and a direction that always points from the character toward the axis, so a
caller can correct in one step without checking a sign.

## `DToPlain` / `DDToPlain`

**Contract** — the offset resolved onto the normal alone: how far the character is out from
the ladder's face. This is the "am I close enough to touch it" measurement.

## `AxDistToUpperP` / `AxDistToLowerP`

**Contract** — signed distances along the axis from the character's feet to each end,
oriented so that both are positive while the character is between the ends and negative
once past. The lower one is negated for exactly that reason, so the two read the same way.

## `InRange`

**Contract** — is the character within the ladder's vertical extent, with a little slack at
each end. The slack is asymmetric: a fifth of a metre below the bottom, nothing above,
except that the character's own foot radius is added to the top test.

**Notes** — the asymmetry is the ladder behaving as a player expects rather than as
geometry suggests. You can stand slightly below a ladder's foot and still be "on" it,
because the bottom rung is usually above the floor; you cannot float above its top,
because the dismount happens there.

## `InTouch`

**Contract** — is the character actually against the ladder: within the face's thickness
plus their foot radius plus a small tolerance, not off the side of the face, and within
the vertical extent. All three must hold. This is the test that decides whether climbing
may begin.

## `BeforeLadder`

**Contract** — is the character *in front of* the climbing face, as opposed to behind it or
inside it. The character's offset from the axis is projected onto the normal and must be
negative past a margin of the ladder's half-thickness plus half a foot radius plus a
caller-supplied tolerance. Unlike `InTouch` this is a half-space test with no range limit,
because it answers "which side am I on", not "am I close".

## `ObjectContactCallback`

**Contract** — the physics world's per-contact filter for this ladder. Identifies which of
the two colliding bodies is the character, rejects the contact outright if neither is,
and otherwise keeps the contact only if that character is in front of the ladder. The
tolerance passed is *negative*, which widens the accepted region slightly beyond the
strict front half-space.

```text
FUNCTION contact_filter(contact, which body is ours) -> keep?
  identify the other body
  IF it is not a character THEN RETURN reject
  ladder = the owning object of our body
  IF NOT ladder.before_ladder(character, tolerance: -0.1) THEN RETURN reject
  RETURN keep
```

**Invariants** — this is what makes a ladder walk-through from behind. A rebuild that
registers a ladder as ordinary static collision will have characters bumping into the
back of every ladder in the game.

**Notes** — the negative tolerance is flagged as suspicious in the original's own comment.
It loosens rather than tightens the test, admitting contacts from characters slightly
*inside* the face. That is probably deliberate — a character being pushed into the ladder
should still collide — but it is not written down.

## `Center` / `Radius` / `UsedAI_Locations` / `register_schedule`

**Contract** — the object's cull bounds are its transform origin and the largest
half-extent. It declares that it occupies no navigation position and that it does not want
to be scheduled: a ladder never updates.

## `DefineClimbState`

**Contract** — empty. An extension point for a ladder that wants to tell a character
something about how to climb it; no ladder does.

## `Load` / `shedule_Update` / `UpdateCL`

**Contract** — pure delegations to the base, present only because the class declares them.

## Could not recover

- The debug orientation helper defined at the top of the file — which would flip the frame
  so its chosen axis agrees with a supplied normal — is never called. Its assertion that
  the thinnest extent and the most-normal-aligned axis are the same axis states an
  invariant the live code relies on but never checks.
- Both extension tolerances are compiled-in: a fifth of a metre below, zero above. The
  lower one is legible as "the bottom rung is off the floor"; the upper being exactly zero
  while the foot radius is added separately looks like two overlapping fixes.
