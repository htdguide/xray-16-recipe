# src/xrGame/flare.cpp

> A hand-held flare: a light that burns for a fixed number of seconds, dims on a fourth-power curve, throws itself away two seconds before it dies, and goes dark.

**Needs** — [`flare.h`](flare.h.md) · [`player_hud.h`](player_hud.h.md) · [`hud_item_object.h`](hud_item_object.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: creates a renderer light and drives it per frame

## Purpose

The whole of the flare. It is short because it delegates everything an inventory item does
to its base class and owns exactly two resources of its own — a dynamic light and a particle
effect — plus a clock. Read it as the minimal example of the chapter's *hud item* shape: a
state machine whose transitions are animation-driven, a per-frame update that runs only
while the item is doing something, and two placement helpers that answer "where is the
business end of this item" differently depending on whether it is being drawn as a
first-person model or as a world object.

## State

```text
RECORD Flare                     # extends hud item
  light_curve    : light_animation   # a named colour-over-time curve from the shared library
  light          : optional<render_light>   # none while not burning
  particles      : optional<particle_effect>
  work_time_sec  : real              # total burn duration, from the item section
```

**Invariants** — the light and the particle effect are created together and destroyed
together; neither exists outside the burning window. `light` being present is the
implementation's test for "currently burning", and the per-frame update does nothing without
it.

The item's *condition* — the generic wear value every inventory item carries, shown in the
inventory screen — is reused here as the remaining fraction of the burn, driven to zero at
burnout. A rebuild should note this is a repurposing, not a coincidence: the flare has no
other notion of wear.

## the state machine

**Contract** — five states; every transition is either commanded from outside or fired by an
animation ending. Entering a state plays that state's animation and marks the item *pending*
(blocking further commands) until the animation ends.

```text
STATE hidden    -> on activate: showing
STATE showing   -> attach to the first-person hands, play the raise animation, light up
                   on animation end: idle, and start the idle animation loop
STATE idle      -> switch the colour curve to the idle one, clear pending
                   this is the only state IsFlareActive reports true for
STATE hiding    -> play the lower animation
STATE dropping  -> play the throw animation
                   on animation end: mark the item as manually dropped, go hidden,
                   and re-enter per-frame processing
```

**Invariants** — the hiding transition guards against re-entering itself: a second request
while already hiding must not restart the animation, or the item never finishes lowering.
No other state carries that guard, because no other state can be commanded twice.

The light is lit on *entry to showing*, not on reaching idle — so the flare is already
burning while it is being raised, which is what the raise animation is authored to look
like.

**Notes** — the drop path ends by *activating* per-frame processing rather than deactivating
it. That is deliberate: the flare has left the hands and becomes a world object that must
keep burning and keep its light placed, and the world-object branches of the two placement
helpers exist exactly for that window.

## `UpdateCL`

**Contract** — the per-frame burn, running only while the light exists. Reads elapsed time
in the current state, ends the burn at the duration, and otherwise updates brightness,
condition, light position and particle transform.

```text
FUNCTION update()
  IF no light THEN RETURN                 # not burning

  elapsed = seconds in the current state
  IF elapsed >= work_time_sec THEN
    extinguish()                          # zero the condition, destroy light and particles,
    RETURN                                # and leave per-frame processing

  IF elapsed + 2 > work_time_sec THEN
    drop()                                # throw it away two seconds before it dies

  brightness = 1 - (elapsed / work_time_sec)^4
  condition  = 1 - (elapsed / work_time_sec)
  colour     = light_curve sampled at the global clock, scaled by brightness
  colour     = white                      # see the note: the line above is overwritten
  light.colour   = colour
  light.position = fire_point()
  particles.transform = particles_matrix()
```

**Invariants** — the elapsed time is measured from entry into the *current state*, not from
ignition. Because the burn spans showing, idle and dropping, and each transition restarts
that clock, the flare's real lifetime is the duration counted afresh in whichever state it
is sitting in. This is a defect worth naming: a rebuild should time the burn from ignition
and keep a separate state clock.

The fourth-power brightness curve is the tuning decision: the flare holds near full
brightness for most of its life and collapses in the last fifth, rather than fading
linearly. The condition, by contrast, falls linearly, so the inventory readout and the
visible brightness deliberately disagree.

**Notes** — the computed colour is discarded: the line after it overwrites it with white,
so the sampled curve and the brightness factor reach nothing. The light-animation lookup,
the two curve names and the colour arithmetic are therefore dead weight in the shipped
build. A rebuild has a choice — reproduce the flat white light the shipped game actually
shows, or restore the intended flicker — and should make it knowingly. Note also the channel
order when sampling: the curve's packed colour is read blue-first into a red-first record,
which suggests the curve data itself is stored in the reversed order.

The auto-drop threshold of two seconds is the throw animation's length plus margin: the
flare must be out of the hands before it goes dark, or the player is left holding a dead
prop.

## `SwitchOn`

**Contract** — creates a shadow-casting point light through the renderer, activates it,
selects the "showing" colour curve, and creates and starts the working particle effect named
by the item's section.

**Notes** — the light type and the shadow flag are held in mutable statics initialised to
"point" and "on". They were a debugging affordance — a place to poke a different light type
from a debugger — and carry no meaning a rebuild should preserve; treat them as constants.

## `SwitchOff`

**Contract** — zeroes the item's condition, destroys the light and the particle effect, and
leaves per-frame processing. Idempotent only in the sense that the caller never reaches it
twice, since the update guards on the light's existence.

## `FirePoint`

**Contract** — the world position of the flame, which is where the light sits.

```text
FUNCTION fire_point() -> position
  IF drawn as a first-person model THEN
    RETURN the hand model's fire-dependency point   # authored per animation
  ELSE
    RETURN the world model's "flare_point" bone, in world space
```

**Invariants** — the two branches must agree closely, or the light visibly jumps at the
moment the flare leaves the hands. The name `flare_point` is a contract with the shipped
model data.

## `ParticlesMatrix`

**Contract** — the full transform for the particle effect, not just a position: particles
need an orientation to emit along.

```text
FUNCTION particles_matrix() -> transform
  IF drawn as a first-person model THEN
    RETURN the hand model's authored particle transform, translated to the fire point
  ELSE
    forward = the object's local +Z in world space
    build an orthonormal basis around forward
    place it at fire_point()
```

**Notes** — the world branch builds its basis from a single direction, so the two remaining
axes are arbitrary but consistent. For a radially symmetric flame that is enough, and it
avoids depending on a bone orientation the model may not supply.

## `UpdateXForm`

**Contract** — deliberately empty, overriding the base class's transform update.

**Notes** — the base class places a held item by attaching it to the carrier's hand bone.
The flare is placed by the first-person hand system instead, so letting the base class also
place it would fight it every frame. An empty override is the C++ way of saying "this item
opts out of the generic attachment"; a rebuild expresses it as a placement strategy chosen
per item rather than as an overridden no-op.
