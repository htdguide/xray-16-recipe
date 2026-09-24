# src/xrGame/Actor_Movement.cpp

> Reconciles what the player asked for with what the body can do: the real movement state, the acceleration the physics receives, the body's facing, and the four predicates that decide whether the actor may run, sprint, jump or move at all.

**Needs** — [`Actor.h`](Actor.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`ActorBackpack.h`](ActorBackpack.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`player_hud.h`](player_hud.h.md) · [`Level.h`](Level.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame state reconciliation feeding a physics body

## Purpose

There are **two** movement states on the actor and keeping them apart is the design.
[`ActorInput.cpp`](ActorInput.cpp.md) fills the *wishful* state — what the player is
asking for. This file computes the *real* state — what the body is actually doing — and
they differ constantly: you asked to sprint but you are limping, you asked to crouch but
the ceiling is too low, you asked to move but a wall is in the way.

Everything downstream — the animation selector, the stamina drain, the interface's motion
icon, the network replication — reads the *real* state. The wishful state exists only
here and in the input layer.

## State

```text
  mstate_wishful : bitset   # the request
  mstate_real    : bitset   # what is happening
  mstate_old     : bitset   # last frame's real state, for edge detection
  model_yaw, model_yaw_delta, model_yaw_dest : real
                            # the body's facing, its strafe-lean offset, and its turn target
  torso, unaffected_torso  : rotation   # aim angles with and without recoil
  landing_time, jump_time, fall_time    : real  # phase timers
  jump_key_pressed         : bool       # latch: prevents auto-repeat jumping
```

### Phase timers

```text
  landing_time_soft = 0.1 s    # a landing that did no damage
  landing_time_hard = 0.3 s    # a landing that hurt
  jump_time         = 0.3 s    # minimum time between jumps
  jump_ground_time  = 0.1 s    # grounded for this long clears the jump state
  fall_time         = 0.2 s    # airborne for this long becomes a fall
```

**Invariant** — falling is *delayed* by a fifth of a second. Without the delay, walking off
a kerb or over a doorsill plays a fall and a landing. The delay is the single most visible
tuning value in the movement system.

## `g_cl_ValidateMState`

**Contract** — the first half of the frame: brings the real state into agreement with
physical reality, before the input is consulted at all. Nine independent rules, run in
order; the order matters where noted.

```text
FUNCTION validate_state(dt, wishful)
  # 1. Leaning: both directions at once cancels; leaning is impossible in the air.
  IF both lean bits set THEN clear lean
  ELSE IF either set THEN copy it into the real state
  ELSE clear lean
  IF jumping, falling or landing THEN clear lean

  # 2. Landing runs down a timer and, on expiry, clears the whole airborne group.
  IF landing THEN
    landing_time -= dt
    IF expired THEN clear landing, fall and jump

  # 3. Touching the ground ends a fall — and chooses WHICH landing.
  IF the movement system reports a ground contact THEN
    IF falling AND the contact speed exceeded 4 m/s THEN
      landing = (the contact cost no health) ? soft : hard
    latch the jump key; re-arm the jump cooldown
    clear fall and jump

  # 4. A jump request released clears the latch. This is what makes jump
  #    a per-press action rather than an auto-repeat.
  IF the jump request is gone THEN clear the jump-key latch

  # 5. Barely moving, or asleep, means not moving — whatever the input said.
  IF (actual speed < 0.2 m/s AND not airborne) OR the body is sleeping THEN
    clear all movement bits

  # 6. Grounded for longer than the ground time guarantees the jump bit clears.
  #    Rule 3 usually does this; this is the backstop for a jump that never left.

  # 7. Touching a wall IS climbing a ladder.
  IF the environment is "at wall" THEN
    IF not already climbing THEN enter climbing, drop sprint, clamp the camera to the ladder
  ELSE
    IF climbing THEN release the camera clamp
    clear climbing

  # 8. Standing up: only when the taller collision box fits.
  IF crouched AND (the crouch request is gone OR climbing) THEN
    IF the standing box can be activated THEN clear crouch
    # a refused stand leaves the actor crouched under the low ceiling

  # 9. Acceleration is revoked when the body cannot accelerate.
  IF cannot accelerate AND the state says accelerated THEN flip the acceleration bit

  # 10. For the controlled actor only: entering or leaving a ladder hides or
  #     un-hides the weapon, under the ladder reason tag.
```

**Invariants** — rule 8 is the pattern for every state change gated on a collision box: the
box swap is *attempted* and the state changes only if it succeeded. A rebuild that changes
the state first and the box after will put the player's head through ceilings.

**Notes** — the contact-speed threshold of four metres per second is what separates a
step down from a fall. The choice between the two landing animations is made on whether the
landing *hurt*, not on how fast it was, so a fall onto a soft material plays the quick
recovery even from a height.

## `g_cl_CheckControls`

**Contract** — the second half: turns the wishful state into the real state and into a
world-space acceleration vector, and computes the jump impulse. Only runs its main body
when the actor is on the ground or on a ladder; in the air the input is recorded but has no
authority.

```text
FUNCTION check_controls(wishful, OUT accel, OUT jump, dt)
  remember this frame's real state as the previous one
  # airborne long enough becomes a fall
  IF airborne AND not already falling THEN
    fall_time -= dt; on expiry set falling and clear jumping

  IF cannot move THEN strip every movement and jump bit from the request

  accel = sum of unit vectors for the four direction bits   # local space

  IF on the ground or at a wall THEN
    crouch:  a new crouch request activates the crouched collision box —
             the accelerated variant if the request is also accelerated,
             the slower one otherwise — and sets crouch only on success.
             Crouching is refused outright while climbing.
    jump:    if allowed and requested, apply the backpack's jump multiplier,
             set the jump state, latch the key, arm the cooldown,
             and spend stamina proportional to the carried-weight ratio

    # Changing speed WHILE crouched swaps the collision box again, because
    # the two crouch gaits have different box heights.
    mask the movement and acceleration bits from the request into the real state
    sprint: granted only if allowed AND the actor is moving forward or strafing
            AND not crouched or climbing AND accelerated

    IF moving THEN
      cancel opposing direction pairs that summed to zero
      scale = base_walk_accel / |accel|
      multiply in, in this order: run or run-backward or walk-backward factor,
        crouch factor, climb factor, sprint factor, strafe factor (run or walk
        variant), the backpack's walk factor, and its overweight factor when
        over the carry limit
      accel *= scale

  IF single player AND the actor just started a new kind of movement THEN
    install a one-shot camera effector named after that movement, if the
    animation file exists, at a strength proportional to the speed scale

  rotate accel from local space into the world by the body's facing
```

**Invariants** — every speed modifier is a *multiplier* applied to one base acceleration,
so the whole movement feel is one authored number and a chain of ratios. The order of
multiplication does not matter mathematically, but the *set* does: a sprinting strafe is
impossible (the sprint grant forbids it), so the sprint and strafe factors never compose.

**Notes**

- Cancelling opposing directions after the vector has been summed, rather than at input
  time, is what makes "forward plus back" mean *stand still* rather than *walk forward*.
  The input layer sets both bits; only here do they annihilate.
- The movement camera effector's strength divides the speed scale by seventy, a constant
  with no derivation. The effector is looked up by file name under a fixed directory, and
  a missing file silently means no effect — which is how a modification adds a per-gait
  camera sway by dropping in a file.
- The camera effector is installed on the *controlled* actor, fetched again by a cast that
  asserts. In single player that is always this actor.

## `g_Orientate`

**Contract** — sets the body's world transform. The body does not face exactly where the
camera looks while strafing: it is yawed by an authored angle so that a strafing walk
animation reads as a body moving sideways rather than a body sliding. Six authored angles
cover the six diagonal and pure-strafe combinations; the offset is eased toward its target
rather than snapped.

The same call eases the lean's target roll toward its authored quarter-turn, and cancels it
when both lean directions are held.

**Invariants** — on a ladder the ladder orientation wins outright and the strafe offset is
skipped.

**Notes** — the six angles are read once from a configuration section and cached for the
process's lifetime, so they cannot be retuned at run time. The lean angle is a compiled-in
constant rather than configuration, which is inconsistent with the strafe angles being
data.

## `g_LadderOrient`

**Contract** — faces the body into the ladder. Takes the ground normal the movement system
reports; refuses when that normal is more vertical than forty-five degrees (that is a
floor, not a ladder) or is degenerate. Otherwise builds an orthonormal frame whose forward
axis is the inverted normal — into the wall — with world up as the reference, and writes it
as the body's transform, preserving position.

**Notes** — the frame's right axis is inverted after construction to correct the handedness
the cross-product order produces. A commented-out earlier version interpolated toward the
ladder frame instead of snapping; the shipping code snaps, and the camera's own easing in
[`ActorCameras.cpp`](ActorCameras.cpp.md) is what hides it.

## `g_cl_Orientate` · `g_sv_Orientate`

**Contract** — the two ways the torso's aim angles are set: from the local camera, or from
the authoritative network state. Both maintain the *unaffected* angles — the aim before
weapon recoil is added — because recoil must not accumulate into the player's aim.

```text
client:  torso = active camera's world yaw and pitch
                 (the first-person camera's, when in free look — free look
                  moves the view without moving the aim)
         unaffected = torso
         IF the active weapon is in its second fire mode AND the view is not
            first-person THEN add the last recoil delta on top

         IF moving THEN the body snaps to the aim, and any turn-in-place ends
         ELSE the body turns to the aim only once the aim is more than
              45 degrees away, then eases to it, clearing the turn on arrival

server:  body facing comes from the replicated state
         torso = the unaffected angles as replicated
         the same second-fire-mode recoil is added, in all three axes
```

**Invariants** — the forty-five-degree dead zone is why a standing player can look around
without the body spinning, and why the body then turns as one motion rather than tracking.
It is also why the turn-in-place animation exists at all.

**Notes** — the recoil addition is gated on the *second* fire mode, which in this game's
data is burst fire, and on the camera not being first-person — in first person the recoil
is applied to the camera itself (see
[`ActorCameras.cpp`](ActorCameras.cpp.md)) and adding it here would double it. The server
path applies it unconditionally, with the camera check left in as a comment: an asymmetry
between the two paths that a rebuild should resolve deliberately.

## `isActorAccelerated`

**Contract** — the single predicate everything uses to ask "is this state a fast one". Its
body is the most surprising thing in the file: the acceleration bit means **walk**, and its
absence means **run**. The game's default gait is running and the accelerate action slows
you to a walk.

```text
FUNCTION is_accelerated(state, aiming) -> bool
  fast = NOT (state has the acceleration bit)
  IF crouched, climbing, jumping or landing THEN RETURN fast   # aim and lean do not apply
  IF leaning OR aiming THEN RETURN false                        # never fast while aiming
  RETURN fast
```

**Notes** — the inverted sense is a genuine trap for a rebuilder: the bit is named for the
*key*, and the key is "walk slowly". Aiming down the sights forces the slow gait
unconditionally, which is a gameplay rule rather than a physical one.

## `CanAccelerate` · `CanRun` · `CanSprint` · `CanJump` · `CanMove`

**Contract** — the five permission predicates, each an AND of unrelated conditions. Read
together they are the complete list of things that can stop the player moving:

- **accelerate** — not limping, not dragging a physics object, and past a timed lock
  another system can set.
- **run** — not aiming down the sights and not leaning.
- **sprint** — everything accelerate requires, plus not stamina-exhausted, plus the game
  mode's permission, plus able to run, plus not strafing, plus the inventory's permission
  (a two-handed object being carried forbids it), plus the sprint-block counter at zero.
- **jump** — not dragging an object, not already jumping, past the jump cooldown, the jump
  key not still latched, and not aiming.
- **move** — not stamina-exhausted and not over the walking weight limit; in either case an
  explanatory message is shown *only if the player is actually trying to move*. Also false
  while in a conversation.

**Invariants** — the sprint-block counter being a counter rather than a flag is the one
non-obvious piece; see [`Actor_Events.cpp`](Actor_Events.cpp.md).

**Notes** — showing the "cannot walk" message from inside a predicate is a side effect in a
query. A rebuild should return a reason and let the caller present it.

## `MaxCarryWeight` · `MaxWalkWeight` · `get_additional_weight`

**Contract** — the two weight limits, each the base value plus the same bonus. The bonus is
the sum of the outfit's, the backpack's, and every artefact on the belt — each artefact's
contribution scaled by its own condition, so a worn artefact carries less.

**Invariants** — only belt artefacts count. An artefact in the rucksack is dead weight,
which is the mechanic.

## `StopAnyMove` · `is_jump`

**Contract** — clearing both movement states at once (and telling the first-person weapon
model the movement ended, for the viewed actor), and the query for whether the actor is in
any airborne phase.
