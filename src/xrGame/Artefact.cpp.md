# src/xrGame/Artefact.cpp

> The artefact: an object that glows and hums while it lies in the world, grants passive effects while it is carried, can be activated into an anomaly, and — for some of them — hides from the player and wanders a patrol path until a detector finds it.

**Needs** — [`Artefact.h`](Artefact.h.md) · [`hud_item_object.h`](hud_item_object.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [`artefact_activation.h`](artefact_activation.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`restriction_space.h`](../xrServerEntities/restriction_space.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame light and particle updates plus a physics-step participation

## Purpose

The base class every artefact derives from, and the largest single item class in the
chapter. Read it as four independent mechanisms sharing one object:

1. **presentation while loose** — a coloured light and a particle effect that follow the
   artefact and switch off when it is picked up;
2. **the carry effect** — five restoration rates and a per-damage-type immunity set that the
   owner's condition reads;
3. **activation** — a carried artefact can be *used up* to spawn an anomaly, which is a
   multi-second process with its own animation and its own physics participation;
4. **detector support** — the newest game's artefacts are invisible by default, drift along
   an authored patrol path, and become visible only when a detector passes close.

The fourth is the one that justifies the file's size, and it is entirely optional per
artefact section.

## State

```text
RECORD Artefact
  particles_name        : optional<text>
  lights_enabled        : bool
  trail_light_colour    : colour
  trail_light_range     : real
  trail_light           : light handle          # created on demand; null while carried
  health/radiation/satiety/power/bleeding restore speeds : real   # the carry effect
  hit_immunities        : per-damage-type factors
  can_spawn_zone        : bool                  # whether activation is possible at all
  af_rank               : int = 0               # which detector grade can see it
  additional_weight     : real                  # added to the owner's carry limit while on the belt
  carrying_bone         : bone index            # multiplayer only: where it rides on the carrier
  activation            : optional<Activation>  # created lazily on first activation
  detector_support      : optional<DetectorSupport>   # created at spawn if configured

  render_frame          : int      # the last frame this artefact was drawn in
  fast_mode             : bool     # whether it updates every frame or only when scheduled
```

**Invariant** — the light exists *if and only if* the artefact is loose, visible and its
section enables lights. Every path that changes ownership or visibility starts or stops it,
and the start path asserts the light is not already there.

**Invariant** — every light and particle operation asserts that the physics world is **not
mid-step**. Creating or destroying a renderer resource from inside the solve is the failure
this whole family of assertions guards against, and it recurs across the chapter.

## The fast/slow update decision

**Contract** — an artefact chooses each scheduled update whether it wants per-frame work.

```text
FUNCTION schedule_update(dt)
  IF carried THEN slow                              # a carried artefact has no light or particles
  ELSE
    drawn_this_frame = (the current frame equals the last render frame)
    distance = |camera - centre| - radius
    IF drawn_this_frame OR distance < 50 m THEN fast ELSE slow

  IF slow THEN do the per-frame work HERE, once, at the scheduled rate
  IF loose AND detector support exists THEN advance the detector support
```

**Invariants** — the work is done either in the per-frame path or in the scheduled path,
never both. That is the whole point of the flag: an artefact fifty metres away and off
screen still updates, just rarely.

**Notes** — the fifty-metre threshold is a compiled-in constant. "Drawn this frame" is
observed by the renderer stamping a frame number on the object, which is a frame *late* —
the artefact learns it was visible after the fact. Harmless for a light's position, and a
rebuild with a visibility query can do better.

The artefact requests a tight scheduling interval (roughly 20 to 50 milliseconds), because
its light position must not visibly lag.

## `Load`

**Contract** — reads the particle name, the light colour and range (only when lights are
enabled), the five restoration rates, the per-damage-type immunity table, the activation
capability, the detector rank and the carry-weight bonus.

**Invariants** — the immunity table's *sense is inverted between the oldest game and the
two newer ones*, and the loader converts on read using the game-material library's version
as the generation signal. The same generation problem appears in
[`ActorHelmet.cpp`](ActorHelmet.cpp.md) with a different heuristic; a rebuild should
determine the generation once, at startup, and pass it everywhere.

**Notes** — the activation capability is decided by whether the artefact's section appears
as a *key* in a dedicated configuration section. That is a membership test against a table
rather than a flag on the artefact, which keeps the list of activatable artefacts in one
place in the data.

## `net_Spawn` · `net_Destroy`

**Contract** — spawn creates the detector support when the section says the artefact can be
controlled, then spawns the base object, starts the particles and the light, plays an idle
animation if the model has one, and puts the artefact in the hidden state. Destroy stops the
lights, destroys the light handle, leaves the physics world's participant list, and deletes
both optional sub-objects.

**Invariants** — the detector support must exist *before* the base spawn, because the base
spawn can make the object visible and the support owns visibility. The fast-mode flag is
initialised to false with a comment claiming the opposite; the first scheduled update
corrects it either way.

## `OnH_A_Chield` · `OnH_B_Independent`

**Contract** — the two ownership transitions, and they are exact mirrors.

Becoming owned: stop the light; in single player stop the particles entirely, and in
multiplayer instead resolve the carrier's head bone so the artefact can be drawn riding on
them; clear any patrol path in progress.

Becoming loose: start the light and the particles again.

**Invariants** — the single-player and multiplayer branches differ because only multiplayer
shows a carried artefact on the carrier's body. See
[`GraviArtifact.cpp`](GraviArtifact.cpp.md) for the other half of that.

## `UpdateWorkload`

**Contract** — the per-frame work, called from whichever path is active.

```text
FUNCTION update_workload(dt)
  REQUIRE the physics world is not mid-step

  # Particles must inherit the carrier's velocity, or a trail drawn behind a
  # running player is left standing in the air.
  velocity = the parent's linear velocity, or zero
  set the particle system's parent velocity

  move the light to the artefact's position

  IF an activation is in progress THEN
    join the physics world's participant list
    advance the activation
    RETURN                        # an activating artefact has no other behaviour

  IF the artefact is not currently attached to a bone THEN
    run the subclass's own per-frame hook
```

**Invariants** — the subclass hook is suppressed while the artefact is attached, which is
what stops a hovering artefact from trying to hover while riding on someone's back.

## activation

**Contract** — four entry points around a lazily created activation object.

- **`ActivateArtefact`** — asserted to be called only on an activatable, carried artefact.
  Creates the activation object if absent and starts it.
- **`PhDataUpdate`** — the artefact's participation in the physics step; forwarded to the
  activation only while one is running.
- **`StopActivation`** — cancels it.
- **`CanTake`** — false while an activation is in progress, so an artefact cannot be picked
  up mid-transformation.

**Notes** — the activation object is created against the *current* parent's identifier and
never recreated. An artefact activated, cancelled, traded and activated again would spawn
its anomaly attributed to the first owner. A rebuild should recreate it per activation.

## The first-person item states

**Contract** — the artefact is also a held item with its own small state machine: hidden,
showing, idle, hiding, and one extra state, *activating*.

```text
Action(fire):  press   -> enter activating, if the artefact can spawn a zone
               release -> return to idle, if currently activating
                          # releasing early CANCELS: activation requires a held button

state entered:  showing    -> play the show animation
                hiding     -> play the hide animation (only if not already hiding)
                activating -> play the activation animation
                idle       -> play the idle animation

animation ended: hiding     -> become hidden
                 showing    -> become idle
                 activating -> on the authoritative side: start hiding AND send
                               the activate-artefact event at the owner
```

**Invariants** — the activation is committed by the *animation ending*, not by the button
release. Holding the button through the whole animation is the input, and letting go early
is the cancel. The commit is an event addressed to the owner, which is how the owner's own
handler (see [`Actor_Events.cpp`](Actor_Events.cpp.md)) performs it.

## `OnActiveItem` · `OnHiddenItem`

**Contract** — becoming the held item plays the show animation and then forces the state to
idle; being put away plays the hide animation in single player, or skips straight to hidden
in multiplayer, and forces the state.

**Notes** — both set the state *twice*, once by switching (which plays an animation) and
once by forcing. The force is what makes the state correct immediately for anything that
reads it this frame, while the animation catches up. It is a workaround for the state
machine being animation-driven; a rebuild with an explicit state and a separate animation
does not need it.

## `UpdateXForm`

**Contract** — places a carried artefact in the owner's hands. Computes a frame from the two
weapon-attachment bones — forward along the line between them, right from the primary bone's
up vector — and composes the item's authored offset onto it. Cached per frame, skipped
entirely when the artefact is attached to a bone by the attachment system instead.

**Invariants** — deriving the orientation from *two* bones rather than one is what makes the
artefact sit in the grip rather than at a single bone's arbitrary rest orientation. The same
construction appears for every held item in the chapter.

## `Interpolate`

**Contract** — on a client, discards every queued network state but the newest and applies
that. An artefact is explicitly **not** interpolated: it is small, it is usually carried,
and a one-update snap is invisible.

## `MoveTo` · `ForceTransform`

**Contract** — teleport the artefact, writing the transform through the physics body as well
as the object, so the solve does not immediately undo it. `MoveTo` is a no-op when the
artefact has no body.

## `create_physic_shell`

**Contract** — builds the body and **immediately deactivates it**. An artefact's body exists
from spawn but is inert until something wakes it, which is what lets a loose artefact lie
still without costing solver time.

## `SwitchAfParticles` · `StartLights` · `StopLights` · `UpdateLights`

**Contract** — the presentation switches. The light's shadow-casting is a per-section
option, defaulting off — an artefact that casts shadows costs a shadow map.

## `SArtefactDetectorsSupport` — the hidden, wandering artefact

This is the newest game's artefact behaviour and it is a self-contained mechanism.

### `SetVisible`

**Contract** — appears or disappears. Records the moment of the switch; returns early if
already in the requested state *after* recording it, so repeated requests keep refreshing
the timer. Appearing starts the light, plays a reveal particle effect on an authored bone
and a reveal sound; disappearing only stops the light. Either way the object's visibility
and its idle particles follow.

**Notes** — the code reads a *hide* particle name and a *hide* sound in an unreachable
branch: the conditional selecting between the show and hide names is nested inside a branch
that already required "show". The hide effects can never play, and a rebuild should either
wire them up or drop the keys.

### `UpdateOnFrame`

**Contract** — three independent rules, each on its own schedule.

```text
# 1. Drift along the patrol path, while hidden.
IF a patrol path is set AND the artefact is invisible THEN
  IF within 2 m of the destination THEN pick a RANDOM neighbouring vertex as the next
  direction = toward the destination
  IF the artefact is moving slower than 0.7 m/s
     OR its motion is more than 45 degrees off that direction THEN
    apply an upward-biased acceleration along the direction
    # applied as GRAVITY, not as an impulse: it is a steady drift, not a shove

# 2. A revealed artefact hides itself again after 5 seconds —
#    but only if its rank is non-zero. Rank-zero artefacts, once found, stay found.
IF visible AND rank != 0 AND 5 s since the visibility switch THEN hide

# 3. A hidden artefact BLINKS every two hours of game time, if the player is
#    far enough away not to be looking at it.
IF invisible AND 12 minutes of real time since the last switch THEN
  reset the timer
  IF the camera is more than 40 m away THEN play the reveal particles for 1 second
```

**Invariants** — the random neighbour choice makes the drift a random walk on the patrol
graph rather than a circuit, so an artefact does not return to a place the player has
already searched.

**Notes** — the two-hour interval is written as a game-time duration divided by ten and then
compared against the *real-time* global clock, so the actual period is twelve real minutes
regardless of the time factor. Whether that mismatch is intended is not recoverable; the
comment says "2 hours of game time" and the code does not implement it.

The blink exists to give a patient player a chance of spotting an artefact without a
detector, and it is suppressed nearby precisely so it cannot be seen happening.

### `FollowByPath`

**Contract** — attaches the artefact to a named patrol path at a starting vertex, with a
drift force. A path that does not exist leaves the artefact stationary.
