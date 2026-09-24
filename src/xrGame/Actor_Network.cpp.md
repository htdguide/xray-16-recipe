# src/xrGame/Actor_Network.cpp

> The actor's whole lifecycle plus its replication: what a spawn record becomes, what the wire carries each tick, how a remote player's motion is smoothed into a plausible path, how a save is written, and what a death reports.

**Needs** — [`Actor.h`](Actor.h.md) · [`Actor_Flags.h`](Actor_Flags.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`Inventory.h`](Inventory.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`Level.h`](Level.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`map_manager.h`](map_manager.h.md) · [`actor_memory.h`](actor_memory.h.md) · [`actor_statistic_mgr.h`](actor_statistic_mgr.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`game_base_kill_type.h`](game_base_kill_type.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: quantized wire encoding with fixed bit widths, and per-physics-step state capture

## Purpose

This is the file the chapter opener's lifecycle section describes in the abstract. It holds
the actor's spawn, its destroy, its reference-release hook, its save and load, its network
export and import, its prediction-and-interpolation machinery, and the death reporting that
produces a multiplayer kill feed.

The interpolation is the substantial part and it is worth stating plainly: a remote player's
position arrives a few times a second, and the client must produce a smooth path through
those points that also respects the physics the client is simulating locally. The engine
does that by running the physics forward past the known state, keeping both the recalculated
and the predicted results, and drawing a **cubic curve** through them.

## State

```text
RECORD NetUpdate                     # one received authoritative sample
  timestamp   : int (32-bit, server clock in milliseconds)
  position    : vector
  model_yaw   : real                 # the body's facing
  torso       : rotation             # the aim, BEFORE recoil
  movement_state : bitset (16-bit)   # only the low 16 bits of the real state cross the wire
  accel, velocity : vector           # each sent as a compressed direction

RECORD InterpolationEndpoint
  position, velocity : vector
  model_yaw, torso   : angles

  NET       : queue<NetUpdate>       # invariant: at most 5, newest last, monotonic timestamps
  NET_A     : queue<PhysicsState>    # the same for the single-body physics state
  last_state, recalculated_state, predicted_state : PhysicsState
  SCoeff[3][4], HCoeff[3][4] : real  # two cubic curves through the same endpoints
  interp_start_time, interp_end_time : int
```

**Invariant** — an update older than the newest already queued is dropped outright, and one
with an identical timestamp *replaces* it. Out-of-order delivery must never move the actor
backwards.

**Invariant** — the movement state is truncated to sixteen bits on the wire, so any bit
above that is local-only. A rebuild adding a movement bit must check which half it lands in.

## `net_Export`

**Contract** — writes this actor's authoritative state. Two shapes, chosen by whether the
actor is alive.

```text
alive:  health, server timestamp, flags byte, position,
        body yaw, and the THREE UNAFFECTED torso angles       # recoil is not replicated
        team, squad, group
        movement state (low 16 bits), saved acceleration (compressed direction),
        velocity (compressed direction), radiation, active slot index
        then a body count, and if non-zero the single rigid body's full state:
        enabled flag, angular and linear velocity, force, torque, position,
        and the orientation as four raw floats

dead:   the same header, then the ragdoll's every bone (see below)
```

**Invariants** — the exported aim is the *unaffected* one, the aim before weapon recoil.
Recoil is reproduced independently on each machine from the shared random seed (see
[`Actor_Events.cpp`](Actor_Events.cpp.md)); replicating the post-recoil aim as well would
apply it twice.

The body count is forced to zero when the actor is carried by a parent, in single player,
or when a client somehow has more than one body — a client never exports a ragdoll.

## `net_ExportDeadBody`

**Contract** — a ragdoll is thirty-odd rigid bodies and sending each one's position as three
floats is prohibitively large. Instead the exporter computes a **bounding box over every
bone's position and over its position-plus-a-tenth-of-its-velocity**, sends that box as two
uncompressed vectors, and then sends each bone as bytes quantized within it.

```text
FUNCTION export_dead_body()
  min, max = bounds over, for every bone: its position, and position + velocity/10
  write a fixed marker byte, then min and max
  FOR EACH bone
    write position, quantized to a byte per axis within [min,max]
    write orientation, four bytes, each quantized within [-1,1]
    write (position + velocity/10), quantized the same way   # velocity is DERIVED on read
```

**Invariants** — velocity is not sent. It is reconstructed by the reader as ten times the
difference between the two sent positions, which means velocity inherits the position
quantization error multiplied by ten. That is acceptable only because the ragdoll's
velocity is a visual hint, not a simulation input.

The bounds are computed over both quantities precisely so that the derived point is inside
the box and therefore representable.

**Notes** — the quaternion is sent as four independent bytes rather than three plus a
reconstructed fourth; the three-byte form is present in the source, commented out. Four
bytes of independently quantized components do not in general form a unit quaternion, so
the reader clamps and the physics normalizes. A rebuild should send three and reconstruct.

The bounds-updating helper contains a recursive self-call guarded by an assertion that can
never be true. It is dead defensive code; drop it.

## `net_Import` · `net_Import_Base` · `net_Import_Physic`

**Contract** — the mirror of the export, plus the queueing discipline. Health, radiation and
the active slot are applied *immediately on a client* rather than queued, because they are
not interpolated. The movement sample and the physics sample go into their own bounded
queues.

**Invariants**

- A **locally controlled actor on a client ignores its own replicated state** entirely and
  returns before queueing. The local player is authoritative over its own motion; the
  server's copy is used only for reconciliation elsewhere.
- Queue depth is five. An older sample is dropped; an equal-timestamp sample replaces.
- The roll angle is wrapped from the unsigned range the wire carries into a signed one.
- During demo playback the received aim drives the camera directly, which is how a recorded
  match is watched from the recorded player's eyes.

**Notes** — the ragdoll import reconstructs the linear velocity from the two quantized
points as described above, and seeds both the current and the *previous* position and
orientation with the same value, so the first physics step after an import does not see a
spurious delta.

## The prediction and interpolation cycle

Four hooks, called by the physics world around its own stepping, in this order every
physics frame:

### `PH_B_CrPr` — before correction-prediction

**Contract** — capture where we are, then jump the simulation to where the server said we
were.

```text
IF already activated this frame, or too many steps have passed THEN RETURN

IF alive THEN
  record the CURRENT position, velocity, body yaw and unaffected aim as the
    interpolation START point
  capture the rigid body's state as "last state"

  IF this is our own actor on a client THEN
    un-freeze the body and set it to the newest received state
      # our own reconciliation: snap to the server, then re-simulate forward
  ELSE
    take the newest movement sample and the newest physics sample
    point the camera at the received aim
    IF the received body was disabled THEN just set the state (it is at rest)
    ELSE
      un-freeze, set the state, run one movement step with the received
      acceleration, then RESTORE the position to the start point
        # the step is run for its SIDE EFFECTS on the movement system's
        # internal state, not for the position it produces
ELSE   # dead: a ragdoll
  IF the bone count does not match what we received THEN RETURN
  un-freeze and write every received bone state, forcing each enabled
```

### `PH_I_CrPr` — between prediction steps

**Contract** — capture the state the physics produced from the *received* input. This is
the "recalculated" state: where the server's sample, re-simulated on this client, actually
puts the actor.

### `PH_A_CrPr` — after prediction

**Contract** — capture the state after the extra prediction steps ("predicted"), then
restore the body to the recalculated one, adopt the received movement state, and build the
interpolation curves. Restoring rather than keeping the predicted state is the point: the
prediction is used as a *curve endpoint*, not as the actor's position.

### `CalculateInterpolationParams`

**Contract** — builds two cubic curves from three known quantities: where the actor was
(start), where it is after re-simulating the server's sample (recalculated), and where it
will be after prediction (end). The curve is evaluated over the interval between the last
update and a computed end time.

```text
start point  = the captured pre-correction position
end point    = the predicted position
start slope  = the current velocity, or — when mid-interpolation already —
               the analytic derivative of the previous curve at the current
               parameter, so successive curves join SMOOTHLY
end slope    = the predicted position minus its previous position, per step

# Clamp each tangent so it cannot exceed a third of the total path length.
# Without this an overshooting velocity makes the curve loop back on itself,
# which reads as a remote player stuttering backwards.
IF |start slope| > total_length/3 THEN rescale it to exactly that
IF |end slope|   > total_length/3 THEN rescale it to exactly that

# Two parameterizations of the same four control points:
SCoeff = the Bezier form over four control points
HCoeff = the Hermite form over two points and two tangents

duration = (one physics step - the frame's physics time) + interpolation steps * step
interp_start = the last update's arrival time; interp_end = start + duration
```

**Invariants** — three interpolation modes exist and are selected by a runtime variable:
linear between the endpoints, the Bezier curve, or the Hermite curve. All three are
computed every frame and only one is applied. A rebuild should compute one.

**Notes** — the derivative formula for the Bezier form is divided by three, with a comment
saying the formula's speed is three times the speed used when the coefficients were
computed. That factor is a genuine correction, not a fudge, and dropping it makes remote
players move at triple speed for a frame after each update.

A commented-out branch would have shortened the interpolation window when the endpoint
velocities were high. It is disabled; the window is always the constant duration.

### `make_Interpolation`

**Contract** — evaluates the curve at the current server time and writes the result into the
movement system as a position *and* a velocity, and points the camera at the interpolated
aim. Past the end time, interpolation stops: the actor snaps to the predicted state and
adopts the received movement state as both real and wishful.

**Invariants** — the aim angles are interpolated as *angles* (shortest arc), the position as
a curve, and the velocity as the curve's derivative. Feeding the movement system a velocity
as well as a position is what keeps the remote player's animation in step with its motion;
a position alone would produce a sliding figure playing an idle.

## `net_Spawn`

**Contract** — builds a live actor from its spawn record. This is the chapter's canonical
spawn order and the order is load-bearing throughout.

```text
FUNCTION spawn(record)
  1.  reset the holder identifier, the touch-character count, the noise level
  2.  tear down any leftover ragdoll shell
  3.  on the authoritative side, force the record's "local" flag
  4.  IF the record is both local AND flagged as the player THEN
        install this actor as THE actor — the process-wide singleton
  5.  create the camera effector manager; clear every current-motion field
  6.  initialise the per-actor registries (encyclopedia, news) keyed by entity id
  7.  spawn the inventory-owner half, then the base entity half
        # FAIL propagates: a refused base spawn refuses the actor
  8.  take the money from the record
  9.  seed the movement state from the record, but ONLY the crouch and
      acceleration bits — a saved actor does not resume running
  10. set the collision box for that state; spawn the physics support
  11. IF the record names a holder THEN destroy the walking capsule now
  12. set the body and aim angles from the record; zero the lean
  13. choose the camera mode from a global flag and point it at the record's aim
  14. clear the jump latch and the saved acceleration
  15. enable the object only if it is local (always, in multiplayer)
  16. REGISTER WITH THE SCHEDULER
  17. rebuild everything derived from the visual (see OnChangeVisual)
  18. activate the unconditional per-frame update path
  19. record the current visual as the "no outfit" default
  20. evaluate the skeleton once, so the first frame has a valid pose
  21. IF dead on arrival THEN clear movement and play the death-initial pose
  22. IF the record names a holder THEN register a deferred callback that
      seats the actor once that holder itself spawns
  23. reset the last-hit record
  24. IN SINGLE PLAYER: add two map markers for the player and create the
      statistics manager
  25. mark the actor as reacting to sound; enable the weapon render flags
```

**Invariants** — step 16 is the one the chapter opener names: an object is not alive until
the scheduler knows about it, and it must be registered exactly once. Step 22 is the
solution to a genuine ordering problem — the car a saved player was sitting in may spawn
*after* the player — and the deferred-callback registry exists for that class of problem
alone.

**Notes** — only crouch and acceleration survive a save into the movement state. A player
who saved mid-sprint loads standing. That is deliberate: every other bit describes a
transient the physics would have to agree with.

## `net_Destroy`

**Contract** — the strict teardown, and the order is as load-bearing as the spawn's.

```text
1.  base entity destroy
2.  cancel the deferred holder callback, if one was registered
3.  delete the statistics manager
4.  tell the map manager this object is gone      # it holds markers by identifier
5.  inventory-owner destroy
6.  release the ladder camera clamp
7.  destroy the walking capsule
8.  deactivate and delete the ragdoll shell
9.  physics support destroy
10. delete the sound-shock effector, the debug graph, the camera manager
11. clear the holder reference and identifier
12. clear the belt artefact list and refresh its interface panel
13. clear the default-visual record
14. IF this was THE actor THEN clear the singleton
15. UNREGISTER FROM THE SCHEDULER
16. destroy the camera collision shell, if this actor owns it
```

**Invariants** — this is the order the system requirements' runtime invariant demands: a
destroyed entity must be unreferenced by the scheduler, the render graph and the physics
world before its memory is released. Steps 4, 12 and 15 are the three un-referencings; the
physics teardown at 7–9 precedes them because a live body would otherwise be stepped after
its owner is gone.

**Notes** — a note in the source warns against commenting out the inventory-owner destroy,
which suggests it once was. Skipping it leaks every carried item.

## `net_Relcase`

**Contract** — the *reference release* hook: some other object is about to be destroyed, and
every pointer to it must be dropped now. The actor clears the looked-at object, the
looked-at vehicle, and — if the object being destroyed is the holder the actor occupies —
detaches from it first. Then the base class, the perception memory, the physics support and
the interface each clear their own references.

**Invariants** — this hook is the mechanism by which the engine's no-dangling-reference
invariant is maintained without reference counting. Every subsystem that can hold a
non-owning reference to a game object must implement it, and a rebuild in a language with
weak references can delete the whole mechanism — which is the single largest simplification
available in this chapter.

## `net_Relevant`

**Contract** — whether this actor's state is worth exporting at all. On the authoritative
side: yes if it is server-updated or locally owned. On a client: yes only if locally owned
and alive.

## `SetCallbacks` · `ResetCallbacks`

**Contract** — install or remove the four aim-bend callbacks on the spine, upper spine,
shoulder and head bones. The bone names are literals and are part of the frozen model
convention: every actor visual must have this skeleton naming.

## `OnChangeVisual`

**Contract** — everything derived from the model, rebuilt. Called at spawn and whenever the
outfit changes the visual.

```text
1.  detach the ragdoll shell across the base class's own visual change,
    then reattach it      # the base class would otherwise free it
2.  reload the footstep table for this section
3.  install the bone callbacks
4.  build every motion table (see ActorAnimation)
5.  build the vehicle steering tables
6.  reload the damage-per-bone table
7.  resolve the named bones: head, both eyes, the three weapon-attachment
    bones (named in configuration, not hard-coded), neck, both clavicles
    and the three spine bones
8.  re-attach every attached item, since the bones they hang from moved
9.  physics support visual change
10. install the bone callbacks AGAIN
11. invalidate every current-motion field, so the next frame re-selects
```

**Notes** — the callbacks are installed twice, before and after the physics support's own
visual change. The second call is the necessary one; the first is defensive. The
weapon-attachment bones are configuration-named while every other bone is a literal, which
is an inconsistency a rebuild should resolve in favour of configuration.

## `ChangeVisual`

**Contract** — swaps the model. No-op when the name is empty or unchanged. After the swap,
re-selects the animation for the current movement state and forces a full skeleton
evaluation, so the new model is posed before it is first drawn rather than appearing for
one frame in its bind pose.

## `save` · `load` · `net_Save`

**Contract** — three different serializations and they are not interchangeable.

- **save / load** is the game save: the base entity, the inventory owner, the
  out-of-bounds flag, four interface filter toggles from the task screen, and the four
  quick-use slot names.
- **net_Save** is the entity's authoritative record: the base entity's, the physics
  support's, and the occupied holder's identifier.

**Invariants** — the quick-use slot names are process-global strings, not actor fields, and
saving them here is the only reason they survive a reload. A rebuild should own them on the
actor.

**Notes** — reaching into the task screen from the actor's save path couples persistence to
the interface; if the screen does not exist, the flags save as false and silently reset. A
rebuild should hold these as game state and let the screen read them.

## Death reporting

Four entry points, all inert in single player and all authoritative-side only. Each emits a
game event the multiplayer game mode turns into a score and a kill-feed line.

- **`SetHitInfo`** — records who hit, with what, on which bone, from which direction, and
  the health before the hit. Everything below reads this record.
- **`OnHitHealthLoss`** — reports a non-fatal hit with the amount of health lost.
- **`OnCriticalHitHealthLoss`** — reports a fatal hit, and classifies it:

```text
kind = none
IF the weapon was a knife THEN kind = knife kill
IF the bone hit was the head AND the weapon was a firearm THEN
  kind = headshot; ALSO broadcast a headshot-particle event
ELSE IF the bone was either eye THEN kind = eye shot
ELSE walk the bone's PARENT CHAIN; if it reaches the head, kind = headshot
IF the back-stab flag is set THEN kind = backstab     # overrides everything
```

- **`OnCriticalWoundHealthLoss`** and **`OnCriticalRadiationHealthLoss`** — death by
  bleeding and by radiation, with no killer attributed in the radiation case.

**Invariants** — the parent-chain walk is what makes a hit on a helmet, a jaw or a hat count
as a headshot. Without it, only the head bone itself would. The walk terminates because the
root bone's parent is the invalid identifier.

**Notes** — the killer and weapon identifiers are masked to sixteen bits on the wire, which
is exactly the entity identifier's width, so the mask is a no-op documenting the width.
A weapon identical to the killer is sent as zero, so that a melee kill does not report a
weapon.

## `OnPlayHeadShotParticle`

**Contract** — plays the configured headshot particle effect at the hit position, oriented
along the *inverted* hit direction so the spray goes away from the shooter. Does nothing
when no particle is configured. The effect is handed to the persistent layer's
play-and-forget list rather than being owned, because the actor that spawned it is about to
die.

## `Check_for_BackStab_Bone`

**Contract** — the seven bones that count as a back-stab target: head, neck, both clavicles
and the three spine bones. The torso and head, in other words, and not the limbs.

## `BonePassBullet`

**Contract** — whether a bullet passes through a bone rather than stopping. In single player
the base class answers. In multiplayer the outfit answers if one is worn; with no outfit,
the answer is read from a **per-bone parameter stored on the skeleton itself**, compared
against a half.

**Notes** — reading a gameplay decision out of a skeleton's per-bone parameter slot is the
only instance of that in the chapter, and it exists because an unarmoured multiplayer player
still needs per-bone bullet behaviour and has no armour record to read it from.

## `InventoryAllowSprint`

**Contract** — false when either the active item or the worn outfit forbids sprinting.

## `On_B_NotCurrentEntity` · `ConvState`

**Contract** — the first tells every inventory item it is no longer the viewed entity's, so
they stop rendering in first person. The second formats the movement bitset as a
human-readable string for the debug overlay; it is the definitive list of which bits have
names.

## `Actor()`

**Contract** — the process-wide single actor, asserted to be reachable only in single
player. It is the service-locator pattern the system requirements describe: a rebuild should
pass the actor explicitly, and the recipe notes at each use site what is being reached for.
