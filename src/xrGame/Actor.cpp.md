# src/xrGame/Actor.cpp

> The player's entity: the one object that is simultaneously a living creature, an inventory owner, an input receiver, a camera rig and a conversation partner — and the core of it, construction, tuning, damage, death and the two update paths.

**Needs** — [`Actor.h`](Actor.h.md) · [`Actor_Flags.h`](Actor_Flags.h.md) · [`actor_defs.h`](actor_defs.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`CameraLook.h`](CameraLook.h.md) · [`Artefact.h`](Artefact.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Inventory.h`](Inventory.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`Level.h`](Level.h.md) · [`Hit.h`](Hit.h.md) · [`actor_memory.h`](actor_memory.h.md) · [`location_manager.h`](location_manager.h.md) · [`step_manager.h`](step_manager.h.md) · [`player_hud.h`](player_hud.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: game logic over a character controller; nothing here touches a byte layout, but the per-frame cost is real and the update is split across two rates.

## Purpose

The actor is the single most *mixed* object in the game: it inherits the living-entity
spine and then mixes in input reception, touch and sound senses, inventory ownership,
dialogue management and footstep management. This file owns the parts of it that belong to
no other mix-in — the camera set, the tuned movement and dispersion constants, the damage
pipeline, death, and the split between what runs every frame and what runs on the
scheduler's budget. The other `Actor*` files carve off one concern each (animation,
cameras, input, movement, network, weapons, sleep, vehicle, backpack); the split is real
work-sharing, not arbitrary, because several of those concerns are three hundred lines
each.

One global exists and is load-bearing: **there is exactly one actor**, reachable by name
from anywhere, and almost every subsystem in chapter 23 reaches for it. A rebuild should
treat it as a level-scoped singleton with an explicit lifetime, not as a free variable.

## State

```text
RECORD Actor EXTENDS EntityAlive, InputReceiver, FeelTouch, FeelSound,
                     InventoryOwner, PhraseDialogManager, StepManager

  # --- camera rig -------------------------------------------------------
  cameras          : list<Camera>      # exactly four, indexed by camera style
  cam_active       : CameraStyle       # first-eye / look-at / free-look / fixed-look
  effectors        : ActorCameraManager  # the two effector stacks, per-actor
  bobbing          : BobbingEffector   # created lazily on first scheduled update
  prev_cam_height  : real
  prev_cam_dir     : vector
  ik_cam_shift     : real

  # --- body orientation -------------------------------------------------
  torso            : rotation          # yaw/pitch/roll actually applied
  torso_unaffected : rotation          # the same, before weapon recoil is added
  torso_target_roll: real              # lean, approached over time
  model_yaw        : real              # the visible model's facing
  model_yaw_dest   : real
  model_yaw_delta  : real              # extra offset when strafing while moving

  # --- movement state ---------------------------------------------------
  mstate_wishful   : int (bitset)      # what the input asked for this frame
  mstate_real      : int (bitset)      # what the body actually achieved
  mstate_old       : int (bitset)
  saved_accel      : vector            # last computed acceleration, also the network value
  # invariant: mstate_real is always a subset of what the body's environment permits;
  # it is derived from mstate_wishful and never written by input directly.

  # --- tuned constants, all read from the configuration section ---------
  walk_accel, jump_speed             : real
  run_factor, run_back_factor,
  walk_back_factor, crouch_factor,
  climb_factor, sprint_factor        : real   # multipliers on walk_accel
  walk_strafe_factor, run_strafe_factor : real
  cam_height_factor                  : real
  disp_base, disp_aim                : real   # radians; authored in degrees
  disp_vel_factor, disp_accel_factor,
  disp_crouch_factor,
  disp_crouch_no_accel_factor        : real

  # --- damage bookkeeping ----------------------------------------------
  hit_slowmo       : real              # seconds of acceleration suppression left
  hit_probability  : real              # difficulty-scaled chance a hit registers
  last_hitter_id,
  last_hitting_weapon_id : EntityId
  last_hit_dir, last_hit_pos : vector
  was_back_stabbed : bool

  # --- what the actor is looking at, recomputed each scheduled update ---
  object_looked_at : optional<GameObject>
  person_looked_at : optional<InventoryOwner>
  vehicle_looked_at: optional<Holder>
  inv_box_looked_at: optional<InventoryBox>
  default_action   : optional<text>    # the localized verb the HUD shows

  # --- carried effects --------------------------------------------------
  artefacts_on_belt: list<Artefact>    # mirror of the belt, kept in sync by the
                                       # item-move hooks; the UI panel reads it
  controller_feedback : FeedbackDecay  # see "Controller feedback" below
  holder           : optional<Holder>  # the vehicle or mounted weapon in use
  holder_id        : EntityId
```

**Invariants**

- `artefacts_on_belt` must contain exactly the artefacts currently on the belt and each at
  most once. It is maintained by the four item-placement hooks rather than rebuilt, which
  is why the hooks assert presence and absence; a rebuild that recomputes it per frame
  removes the invariant and the bug class with it, at a small cost.
- When the actor is inside a holder, the actor's own movement, animation and pickup logic
  must not run. The holder drives it instead.
- `mstate_wishful`'s movement bits are cleared at the end of every scheduled update, so a
  held key must be re-observed each update rather than latched. Only the state bits
  (crouch, and optionally sprint) survive, and even crouch is cleared when the toggle
  setting is off.

## `construct` / `destroy`

**Contract** — construction builds the four cameras and loads each one's authored rotation
limits from a named configuration section, installs the default movement tuning, creates
the animation sets, the memory (perception) manager, the location manager and the two
registry wrappers that hold the encyclopedia and news the player has accumulated, and
enables the belt. Destruction tears them down in the reverse order and stops every looping
sound the actor owns.

**Notes**

The *perception manager is not created on a dedicated server*. A dedicated server runs the
game with no rendering and no local player view; the actor still exists as a networked
entity but has nothing to perceive. Several other allocations in this file carry the same
guard, and a rebuild should read them as one decision — "the headless configuration owns
no client-side presentation state" — rather than as a dozen unrelated conditionals.

**The camera set is fixed at four and each has an authored section.** First-eye is the
normal play camera; look-at is a third-person orbit; free-look is an unconstrained orbit;
fixed-look is a stationary camera pointed at the actor, which is what death switches to.
A command-line switch selects an alternative third-person camera and section pair; it
exists because the games differ in whether third person is a supported view at all.

## `Load`

**Contract** — reads the whole actor tuning surface from its configuration section and
pushes it into the parts that need it. Reads: the three collision boxes, crash speeds and
mass for the character controller; the per-restrictor-kind radii (stalker, small stalker,
medium monster) that shrink the actor's collision against friendly traffic; every movement
multiplier; camera height factor and air-control; pickup radius; grenade-sense radius and
delay; the whole dispersion tuning; the hit-sound bank, one list per damage type, plus four
death sounds and three looping condition sounds; the default outfit visual; the invincibility
shield particle names; the auto-pickup probe box; and the head-shot particle.

**Invariants** — the actor declares itself *visible to AI* and *not reacting to sound* in
the spatial database at load, and then turns sound reaction back on during every
per-frame update. The odd pairing is deliberate: the flag must be off while the level is
still assembling so that a half-spawned actor does not receive sound events, and on
thereafter.

**Notes**

**Three collision boxes, not one.** Standing, crouching-and-moving, and
crouching-and-still each get their own box, selected by the movement state. The crouch
boxes differ because a creeping actor is shorter than a crouch-walking one, and the
difference is visible in whether the player fits under a given piece of geometry — this
is gameplay, not an optimization.

**Every box is grown vertically by half the camera-collision shift.** The camera pushes
out of walls by a fixed distance; the collision box has to be at least that much taller or
the camera ends up inside geometry the body is standing clear of.

**Dispersion is authored in degrees and stored in radians.** A rebuild is free to keep
degrees internally; what is frozen is the *authored* unit, because the values come out of
the shipped configuration.

**The scheduler bounds are set to the minimum.** The actor asks for an update every
possible tick and never degrades. It is the one object in the world that may not be
budget-skipped.

**Difficulty is applied here and re-applied on change** — see `OnDifficultyChanged`.

## `Hit`

**Contract** — the actor's damage entry point, and the one place where the damage pipeline
is visible end to end. Validates the damage type, chooses and plays a hit sound, computes
and submits controller vibration, applies artefact protection, offers the script layer a
veto and a mutation of the hit, then hands the result to the living-entity base for the
actual health arithmetic.

```text
FUNCTION Hit(hit)
  IF hit.type NOT IN valid damage types  THEN FAIL WITH "unregistered hit type"

  play_sound = actor is alive
  IF multiplayer AND this player is flagged invincible
     play_sound = false
     spawn the invincibility shield particle at the struck bone,
       choosing the first-person variant when this actor owns the view
       # at most one per frame, hence the frame stamp

  IF a sound exists for this damage type AND the condition system permits it
     pick one at random from that type's bank
     IF explosion AND this actor owns the view
        raise its volume and start the shock effector       # see below
     ELSE IF explosion                                       THEN play_sound = false
     IF play_sound AND no sound of this type is already playing
        play it at the actor's head height
     feedback_duration = max(default, the sound's length)

  submit controller vibration        # see "Controller feedback"

  hit_slowmo = the condition system's slowdown for this hit

  IF this actor owns the view AND the damage is a firearm wound
     raise the directional damage indicator toward the recorded hitter

  IF sprinting AND the condition system says this hit breaks a sprint
     AND it is not a self-inflicted burn
     drop the sprint request

  IF hit marks are enabled AND (firearm wound OR a physically-initiated strike)
     add a directional camera-shake effector                 # see HitMark

  IF single player
     IF god mode  THEN zero the damage and delegate; RETURN
     damage = damage minus the belt artefacts' immunity to this type
     mark the hit as wound-producing
     IF alive
        hand a mutable copy of the hit to the script layer's pre-hit function;
          a false return cancels the hit entirely,
          otherwise power, impulse, direction and type are read back out
        fire the script hit callback
     delegate to the living-entity base
  ELSE
     detect a back stab: a wound-type-2 hit on a back bone whose direction,
       expressed in the actor's own frame, points more than 135° away from forward,
       against a player not flagged invincible
     damage = 100000 if back-stabbed and nonzero, else belt-artefact-reduced
     delegate to the living-entity base
     IF this is the server AND the actor died to an explosion
        record the kill in the weapon-usage statistics
```

**Notes**

**Back-stab is a geometric test, not a flag.** The hit direction is transformed into the
actor's local frame and inverted, and the result must have a forward component below
−0.707 — that is, the attacker is behind the actor by more than 45° either way. The
magic damage value is an instant kill expressed as a number rather than a flag, which is
how it passes through the same arithmetic as any other hit.

**The script veto runs *after* artefact protection and *before* the health change.** That
ordering is part of the script surface's observable behaviour and is frozen by conformance
criterion 10: a script that reads the hit's power sees the artefact-reduced value.

**Controller feedback.** Vibration is two channels — a low-frequency rumble derived from
the hit's physical impulse and a high-frequency buzz derived from its damage. Shock and
radiation produce no rumble at all because they have no impulse; burns produce none below
a damage threshold. An explosion additionally starts a *decaying* feedback: the high
channel is reduced by one step of its own magnitude per fixed interval until the
duration runs out, which reads as a fading ring rather than a hard stop. The decay is
re-submitted from the per-frame update, which is why the record carries both a submit
time and a next-update time.

**The shock effector and the vibration are mutually exclusive.** An explosion loud enough
to start the ear-ringing effector suppresses the plain vibration path, because the effector
drives its own.

## `HitMark`

**Contract** — converts a hit's world direction into one of eight camera-shake effectors
and starts it, scaled by the damage. Runs only for a living, locally-controlled actor that
owns the view, and does nothing if a fire-hit effector is already running.

```text
FUNCTION HitMark(power, direction)
  raise the HUD's directional damage indicator
  IF a fire-hit camera effector is already active  THEN RETURN

  angle = absolute angular difference between the camera's heading and the hit's heading
  above = the cross product of camera direction and hit direction points up

  # eight sectors, but the front and back sectors are twice as wide as the sides
  IF angle <= 22.5°                       id = front
  ELSE IF angle <= 67.5°                  id = above ? front-upper : front-lower
  ELSE IF angle <= 112.5°                 id = above ? upper : lower
  ELSE IF angle <= 157.5°                 id = above ? back-upper : back-lower
  ELSE                                    id = back

  start the effector from configuration section "effector_fire_hit_<id>",
    with strength = power / 1000
```

**Notes** — the sector boundaries are 22.5° then four 45° bands. Front and back get a
narrow dedicated sector and the sides are resolved by the sign of the cross product, which
is how "shot from the left" and "shot from the right" become different shakes without
computing a signed angle. The divide-by-1000 converts a damage figure into the effector's
own unit and is otherwise arbitrary.

## `HitSignal`

**Contract** — plays a one-shot damage flinch animation on the skeleton, chosen by the
struck bone's authored damage-animation index and by whether the hit came from the front or
the back, with the animation's strength scaled from the damage percentage clamped to the
unit range.

**Notes** — the bone carries its animation index as a *parameter on the bone instance*,
which is data authored into the model rather than into the configuration. Front-versus-back
is decided by whether the hit's heading is within 90° of the model's current facing, and
the result selects the adjacent entry in the damage-animation table — so the table is
authored in front/back pairs.

## `Die`

**Contract** — runs the actor's death: on the server, empties the inventory in a specific
order; plays one of four death sounds and stops every looping condition sound; switches the
camera; and clears all movement.

```text
FUNCTION Die(killer)
  delegate to the living-entity base       # health, ragdoll, the death record

  IF this is the server
    FOR EACH slot IN inventory slots
      IF slot is the active slot
         IF it holds a grenade      THEN release the grenade (it becomes live)
         ELSE                       THEN mark the item for manual drop
         CONTINUE                              # the active item is dropped, not stowed
      IF it holds an outfit         THEN CONTINUE     # the corpse keeps wearing it
      move the item to the rucksack
    move every belt item to the rucksack
    IF multiplayer
      mark artefacts for drop in the artefact-hunt mode, and always the player's bag

  IF not a dedicated server
    play one of four death sounds at the body; stop breathing, bleeding and danger loops

  IF single player
    switch to the camera named by the death-camera setting (first-eye, free-look or
      fixed-look); a separate first-person-death switch forces first-eye
    hide every open dialog and start the game-over tutorial sequence
  ELSE
    switch to the fixed-look camera

  clear every movement bit, wished and real
  destroy the shock effector
```

**Notes** — the inventory order is the whole decision: a corpse must be *lootable in the
shape the player expects*, which means everything ends up in the rucksack except the
outfit (still worn, so the corpse looks right) and the active item (dropped at the feet, so
the kill visibly yields a weapon). A live grenade in hand is released rather than dropped,
because dropping it would defuse it.

## `shedule_Update`

**Contract** — the actor's *budgeted* update: everything that need not happen at frame
rate. Runs the control path for a locally-controlled actor and the interpolation path for a
remote one, then the shared tail: motion icon, camera bobbing, condition sounds, visibility,
the look-at probe, artefact effects and the auto-pickup sweep.

```text
FUNCTION shedule_Update(elapsed_ms)
  declare this object server-updated if we are the server

  IF this actor owns the player's view
     attach or detach the active item's first-person model as its hidden state demands,
       and detach everything when the HUD view is off

  IF inside a holder OR disabled OR not ready
     clear the look-at verb; delegate; RETURN      # the holder drives us

  clamp elapsed to 100 ms and convert to seconds

  IF this actor is the one the level's controls drive AND not replaying a demo
     read the controls into an acceleration and a jump impulse
     orient the model and torso from the control state
     run the character controller
     validate the achieved movement state against the environment
     select animations from it
     update the touch sense over a sphere at the actor's centre
     update the grenade sense
     accumulate drop power while the drop key is held
     clear every per-frame movement request     # see the invariant above
  ELSE
     interpolate the networked position
     IF any network update has arrived
        orient from the server's state, run the controller, validate, animate,
        and select the collision box from the server's movement state
     remember the movement state as the previous one

  IF this actor owns the view  THEN update the movement-state icon

  delegate to the living-entity base

  create the bobbing effector on first use, then tell it the movement state,
    whether the actor is limping and whether a sight is raised

  IF this actor is the one the controls drive AND not a dedicated server
     drive three looping sounds from the condition system:
       heavy breathing while limping and alive (suppressed under god mode),
       a blood loop whose volume is bleeding-rate + 0.25 above a rate of 0.6,
       a danger loop whose volume is zone-danger + 0.25 above a danger of 0.1
     stop all three on death

  hide the actor's own model whenever the first-person view is active

  probe what the actor is looking at     # see below
  apply the belt artefacts' and the outfit's condition effects
  run the physics support's scheduled half
  sweep for auto-pickup
```

**Invariants** — the interpolation branch runs *only* when at least one network update has
arrived. A remote actor with no updates yet holds its spawn pose rather than being
extrapolated from nothing.

**Notes**

**The look-at probe reuses the HUD's ray query.** The HUD already casts a ray down the
crosshair every frame for its own reasons; the actor reads that result rather than casting
a second one. A hit within two metres on a visible object becomes the "object we are
looking at", and a chain of type tests turns it into the verb the HUD shows: a custom tip
on the object wins; otherwise a living, talk-enabled character gives *talk*, a corpse gives
one of three verbs depending on whether it is closed, draggable (which is decided by whether
the corpse's *visual model name* appears in a configuration section listing draggable
visuals) or merely lootable, a holder gives its own use verb or the vehicle default, and a
takeable item gives *pick up*. Every verb is a string-table key, never a literal.

**Artefact and outfit effects are applied on a 100-millisecond rhythm, not per update.**
The accumulator is a file-scoped value shared by all actors, which is harmless only because
there is one actor. A rebuild with more than one actor must make it per-actor.

**Belt artefacts restore five condition axes and the outfit restores the same five.**
Radiation is the exception: a *positive* radiation rate (an artefact that irradiates you) is
first reduced by the boost-granted radiation immunity and clamped at zero, while a negative
rate (an artefact that cleans you) is applied unreduced. Immunity protects against being
irradiated and does not cancel decontamination.

**With no outfit and no helmet, night vision is force-disabled.** The torch's night-vision
mode requires a head slot item to hang off; without one it is switched off every update
rather than being prevented from being switched on.

## `UpdateCL`

**Contract** — the actor's *unconditional* per-frame update: everything that must happen at
frame rate because the camera or the first-person model depends on it.

```text
FUNCTION UpdateCL()
  IF alive AND this actor owns the view AND no UI has the input AND not in a holder
     pickup mode is on while any binding of the use action is physically held
                                                    # only when multi-pickup is enabled
  advance the inventory owner's own timers

  IF any touched object is a character
     refresh each touched character's collision box    # they shrink near the actor

  IF inside a holder  THEN let the holder update with our current field of view

  decay the emitted-noise level by 0.3 per second
  delegate to the living-entity base; run the physics support's per-frame half

  IF alive  THEN run the pickup highlight sweep
  run the corpse-search pickup sweep

  clear the aiming flag, then recompute it from the active weapon
  update the cameras with this frame's delta and the current field of view
  tell the device the camera moved

  IF this actor owns the view
     push the weapon's dispersion through the smoothing controller into the crosshair,
       and set the HUD flags the weapon asks for (crosshair, indicators, weapon layer)
     with no weapon, collapse the crosshair and hide it

  deliver any deferred news messages whose delay has expired
  IF alive  THEN advance the footstep manager
  re-enable sound reaction in the spatial database
  advance or destroy the shock effector, destroying it outright if the view moved away
  advance the decaying controller feedback
  build the first-person model's transform from the camera — the HUD camera's matrix
    in first person, the plain camera matrix otherwise — and update the first-person model
  IF multi-pickup is enabled  THEN clear pickup mode
```

**Notes**

**The field of view is a function of the weapon, not a setting.** A zoomed weapon reports
its zoom factor, scaled by three quarters, as the camera's field of view — but only in
first person, only while the rotation into the zoom has finished when the weapon has a
scope texture, and only when the HUD's weapon layer is enabled at all. Everything else uses
the global field of view.

**Pickup mode is edge-shaped by two flags, not one.** With multi-item pickup enabled the
flag is set from the physically-held key at the top of the frame and cleared at the bottom,
so it is true for exactly the span of the frame that does the picking; with it disabled the
flag latches and is cleared by the pickup code itself. The two behaviours are "hold use to
sweep up everything" versus "press use to take one thing".

**The noise level decays at a fixed rate and is raised elsewhere.** It is what the AI's
sound perception reads to decide how audible the player is.

## `g_Physics`

**Contract** — runs the character controller for one step: scales the requested
acceleration by the remaining hit-slowdown, refuses control entirely while climbing with an
unclamped camera, steps the controller, reads the resulting position back onto the object,
and turns the step's two reported outcomes — a ground contact and a collision health loss —
into a landing camera effector and a hit message respectively.

**Invariants** — the level-border crossing is edge-detected here and only for the
controlled actor: entering and leaving the authored playable region fire two different
script callbacks exactly once per crossing.

**Notes**

**Collision damage is sent as a network hit message, not applied directly.** Even in single
player, the actor's fall and crush damage goes out through the event path so that the
server side is the one that applies it. That is the "single player is a networked session
against a local server" rule made concrete, and it is why fall damage has a one-tick
latency.

**A no-clip debug mode skips reading the position back** — the controller still steps, but
the object keeps the position the free-fly code gave it.

**Acceleration is scaled by one minus the remaining hit-slowdown**, which decays in real
seconds. A hit therefore *drags* the player for a configured interval rather than stunning
them for a fixed one.

## `UpdateArtefactsOnBeltAndOutfit`

**Contract** — see the notes under `shedule_Update`; applies every belt artefact's and the
outfit's five per-second condition rates, scaled by the item's condition and by the elapsed
interval, at most ten times a second.

## `HitArtefactsOnBelt` / `GetProtection_ArtefactsOnBelt`

**Contract** — the first subtracts each belt artefact's immunity to a damage type from an
incoming damage figure and clamps at zero; the second sums the same immunities, each scaled
by the artefact's condition, and reports them as a protection figure for the UI.

**Notes** — the asymmetry is real and looks like a bug worth preserving: the *applied*
reduction ignores artefact condition while the *displayed* protection accounts for it. A
worn artefact protects fully and is reported as protecting partially.

## `GetRestoreSpeed`

**Contract** — reports the actor's current net rate of change for one of five condition
axes, summing the condition system's own base rate, the belt artefacts' contributions
(each scaled by condition) and the outfit's. Purely informational — the UI reads it; the
actual changes are applied elsewhere.

**Notes** — power is the exception: after summing, it is *divided* by the outfit's power-loss
factor, or by one half when no outfit is worn. Wearing nothing therefore doubles the power
regeneration rate, which is the cost model that makes heavy armour a real trade.

## `MoveArtefactBelt`, `OnItemTake`, `OnItemDrop`, `OnItemRuck`, `OnItemBelt`, `OnItemDropUpdate`

**Contract** — the item-placement hooks that keep the actor's derived state in step with the
inventory. `MoveArtefactBelt` maintains the belt-artefact mirror and refreshes the artefact
panel when this actor owns the view. Dropping an outfit removes its skin model; dropping a
zoomed weapon un-zooms it and restores night vision if the weapon had suppressed it;
dropping the last grenade from the grenade slot automatically promotes another grenade of
the same kind into that slot. `OnItemDropUpdate` re-attaches any item that has become
invalid and is not currently attached.

**Notes** — the grenade auto-promote is a quality-of-life decision with a gameplay
consequence: the player never has an empty grenade slot while grenades remain, so throwing
is continuous.

## `Load`-time difficulty: `OnDifficultyChanged`

**Contract** — re-reads three difficulty-suffixed configuration sections: the actor's
damage-type immunities, the hit probability, and the two-hits-death parameters. Called at
load and whenever the difficulty setting changes at runtime.

**Notes** — difficulty is expressed entirely as *data selection by name*, not as code
branches: a section name is built by appending the difficulty's token to a fixed prefix.
A rebuild that adds a difficulty level need only add sections.

## `currentFOV`, `Radius`, `GetMass`, `is_on_ground`, `is_ai_obstacle`, `use_center_to_aim`

**Contract** — small derived quantities. The actor's radius is its own plus the active
weapon's, so that a long rifle is included in the touch sense's sphere. Mass comes from the
character controller while alive and from the ragdoll afterwards. The actor is *never* an
AI obstacle — creatures path through the player rather than around them, which is the only
way crowded corridors work. Aiming uses the body's centre rather than its eye only while
crouched.

## `NeedToDestroyObject` / `TimePassedAfterDeath`

**Contract** — in single player a corpse is never removed. In multiplayer it is removed
once a configured time has passed since death, or immediately, or never, depending on a
three-valued setting (−1 never, 0 immediately, otherwise after the body-removal interval) —
and in every case only if the death-removal flag is set.

## Rendering: `renderable_Render`, `renderable_ShadowGenerate`, `OnHUDDraw`, `RenderIndicator`, `RenderText`

**Contract** — the actor draws its body and its attachments; it casts no shadow while
inside a holder (the vehicle's own shadow covers it); it draws the first-person layer
unless a multiplayer lean is in progress. `RenderIndicator` draws a camera-facing quad at
the head bone — the friend/foe marker — and `RenderText` draws a name above the head,
projected to screen space and rejected when behind the camera or off screen.

**Notes** — the name's vertical offset and size are derived from the *projected* height of a
one-unit vertical segment at the head, so a distant player's label lifts clear of the model
and stops shrinking below a floor size. The two constants that set that floor and that lift
have no derivation in the source; they are tuned by eye.

## `SetZoomRndSeed` / `SetShotRndSeed`

**Contract** — install a random seed for the two effects that must agree between a client
and the server: the sight's sway and the shot's camera kick. Passing zero seeds from the
server's current time.

**Notes** — this is the whole of the engine's determinism story for weapon feel in
multiplayer. Both ends generate the same sway and the same recoil because both were handed
the same seed with the shot, rather than by replaying the shot.

## `ForceTransform` / `ForceTransformAndDirection` / `SetPhPosition` / `MoveActor`

**Contract** — teleport the actor. `ForceTransform` moves the body; the direction variant
additionally points the active camera along the new transform's orientation, negating each
angle because the camera's convention is the inverse of the transform's. In multiplayer a
forced transform suppresses collision damage for two seconds' worth of fixed steps,
because a teleport otherwise reads to the controller as an enormous instantaneous
displacement.

## `use_bolts`, `unlimited_ammo`, `use_default_throw_force`, `missile_throw_force`, `spawn_supplies`, `can_attach`

**Contract** — small policy answers. Bolts exist only in single player. Unlimited ammo is a
debug flag. The actor throws with the missile's own default force rather than a
character-specific one. Attachment is permitted only for items whose configuration section
appears in the actor's authored attachable list and only one of each section at a time.

## `_construct` / `create_entity_condition` / `reinit` / `reload`

**Contract** — the two-phase construction the object factory demands: `_construct` runs
before the spawn record is known and builds the physics support and each mix-in's own
pre-spawn state; `create_entity_condition` installs the actor-specific condition object;
`reinit` and `reload` re-run per-spawn and per-section setup across every mix-in.

**Notes** — the ordering inside `reinit` is load-bearing: the character controller must
exist and know its owner *before* the living-entity base re-initializes, because that base
reads the controller's position.

## Module-level values

**Contract** — the actor flag set (god mode, no-clip, auto-pickup, run-backward,
multi-item pickup, tracers and the rest, all console-settable), the look and cursor
sensitivity ranges and steps, the sleep duration, and the four quick-use slot section names.

**Notes** — the flag set's *default* value is itself a decision: god mode is off but
"real-time god mode" is on, auto-pickup, backward running, important-save marking,
multi-item pickup and tracers are on. Those defaults are what an unconfigured install
plays like.
