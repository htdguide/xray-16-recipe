# src/xrGame/Actor_Weapon.cpp

> Where the actor meets a weapon: how far a shot may stray given how the player is moving, where the shot originates, and what recoil does to the view.

**Needs** — [`Actor.h`](Actor.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Missile.h`](Missile.h.md) · [`Grenade.h`](Grenade.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`CameraRecoil.h`](CameraRecoil.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`map_manager.h`](map_manager.h.md) · [`Level.h`](Level.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a scalar formula and a handful of delegations

## Purpose

A weapon knows its own accuracy; the *shooter* modifies it. This file is the shooter's half
of that contract, and the reason it is the actor's rather than the weapon's is that
everything it consults — how fast the player is turning, whether crouched, whether
sprinting, whether aiming — is state only the actor has.

The rest of the file is the recoil effector's lifecycle and three small pieces of
multiplayer convenience.

## State

`Stateless.` The dispersion coefficients are actor fields read from configuration; the
recoil effector lives on the camera stack.

## `GetWeaponAccuracy`

**Contract** — the current dispersion cone half-angle, in radians. Aiming down the sights
short-circuits everything: a fixed aimed dispersion is returned, provided the weapon has
finished raising to the aim. Otherwise a base dispersion is built up from five
multiplicative factors.

```text
FUNCTION weapon_accuracy() -> real
  W = the active weapon, or none

  IF aiming down the sights AND W is not still rotating into the aim THEN
    RETURN the aimed dispersion          # a fixed value; nothing below applies

  d = base_dispersion * W.dispersion_base

  # Each factor is (1 + actor_coefficient * weapon_coefficient * driver).
  # The weapon's coefficient lets one rifle punish movement more than another.
  d *= 1 + (angular_speed / 10) * vel_factor * W.vel_factor
  d *= 1 + (linear_speed  / 10) * vel_factor * W.vel_factor
  IF moving fast OR NOT crouched THEN
    d *= 1 + accel_factor * W.accel_factor
  IF crouched THEN
    d *= 1 + crouch_factor * W.crouch_factor
    IF NOT moving fast THEN
      d *= 1 + crouch_no_accel_factor * W.crouch_no_accel_factor
  RETURN d
```

**Invariants** — the two speed divisors are both ten, a saturation speed above which the
penalty stops growing linearly relative to the coefficient — a player turning faster than
ten radians per second is already inaccurate. Both are compile-time constants with no
derivation beyond feel.

**Notes**

- The accelerated-or-standing branch is the load-bearing one and reads backwards at first:
  the penalty applies when the player is *either* moving fast *or* not crouched, so the only
  way to avoid it is to crouch and move slowly. Crouching then adds its own penalty and a
  further one for crouch-walking, so the *most* accurate stance is crouched and perfectly
  still, which takes the crouch factor but not the crouch-walking one.
- "Moving fast" is the inverted predicate from
  [`Actor_Movement.cpp`](Actor_Movement.cpp.md), where the acceleration bit means *walk*.
  A rebuild must carry that inversion here or invert the whole formula.
- Every weapon coefficient defaults to one when there is no weapon, so a bare-handed actor
  still has a defined dispersion.

## `g_fireParams`

**Contract** — where a shot starts and where it goes: the camera's position and direction,
always. A thrown item is the one exception — its release point is offset from the camera by
an item-specific vector rotated into the actor's frame, so a grenade leaves the hand rather
than the eye.

**Invariants** — firing from the camera rather than from the weapon's muzzle is the decision
that makes first-person aiming exact. It also means a shot can originate inside geometry the
muzzle is clear of, which is why the camera is pushed out of geometry every frame (see
[`ActorCameras.cpp`](ActorCameras.cpp.md)).

## `g_WeaponBones`

**Contract** — the three skeleton bones a weapon is attached to: the right hand and a right
finger for the primary grip, a left finger for the support hand. Resolved from configuration
at visual-change time.

## `g_State`

**Contract** — the snapshot of the actor the weapon and the dispersion formula read: the
four movement booleans (jump, crouch, fall, sprint), the actual linear speed from the
movement system, and the view's angular speed as measured in
[`ActorCameras.cpp`](ActorCameras.cpp.md). Always succeeds.

## `SetCantRunState` · `SetWeaponHideState`

**Contract** — the two ways something asks the actor to restrict itself. Both are sent as
**events**, not applied directly, and both are sent only by the controlled living actor.

**Invariants** — the sprint restriction sends a *delta* of plus or minus one into the
counter described in [`Actor_Events.cpp`](Actor_Events.cpp.md), so several independent
callers can each hold a veto. The weapon-hide restriction sends a reason tag and a boolean,
so each caller un-hides only its own reason. Both are the same pattern: **a restriction is
owned by whoever imposed it and may only be withdrawn by them.**

## `SelectBestWeapon`

**Contract** — after picking something up, switch to it if it is better than what is held.
Multiplayer only. The preference order is fixed: secondary weapon slot, primary weapon slot,
grenade, knife — and the first of those that holds an item ends the search, whether or not
the switch happens.

```text
FUNCTION select_best(picked_up)
  IF single player THEN RETURN                 # the player chooses; the game does not
  IF it is an artefact already owned THEN RETURN
  IF it is not a weapon, grenade or artefact THEN RETURN

  IN the artefact game modes ONLY:
    IF the item's natural slot is already occupied by something else THEN RETURN
      # do not disarm a player who picked up a spare

  FOR EACH slot IN preference order
    IF that slot holds an item THEN
      IF it is not already active AND it can kill THEN activate it
      RETURN                                   # the first occupied slot decides, always
```

**Notes** — the early return inside the loop means the search never falls through to a lower
preference. If the most-preferred occupied slot holds something that cannot kill — an empty
weapon — nothing is activated at all. That is almost certainly not intended and a rebuild
should decide deliberately.

## `HitSector`

**Contract** — puts a marker on the map showing roughly where an attacker is. Suppressed
when the actor is dead, when the attacker is not a living creature or is the actor itself,
and — the interesting rule — when the attacking weapon has a silencer fitted. A silenced
weapon does not give away its shooter's position.

## The recoil effector

**Contract** — five entry points wrapping one camera effector.

- **shot start** — installs the effector if absent, or re-initializes it if the firing
  weapon has changed; picks the weapon's *aimed* or *hip* recoil curve depending on whether
  the player is aiming; seeds it with the shared shot random seed; and fires one shot into
  it.
- **shot update** — advances it and applies it to the camera, each frame from
  [`ActorCameras.cpp`](ActorCameras.cpp.md).
- **shot stop** — tells the effector the trigger was released, so it can recover.
- **shot remove** and **weapon hide** — tear it down, or reset it in place.
- **delta angle** and **last delta** — read the accumulated and the most recent recoil
  offsets, which [`Actor_Movement.cpp`](Actor_Movement.cpp.md) adds to the torso aim in
  third person.

**Invariants** — the effector is keyed by the weapon's entity identifier. Switching weapons
mid-burst re-initializes rather than replacing, which preserves the effector's place on the
camera stack. The random seed is set on *every* shot, not on installation, because it is
re-synchronized from the network per shot.

**Notes** — two recoil curves per weapon, aimed and hip, is the whole model: there is no
interpolation between them, so the transition at the moment the aim completes is a step.

## `SpawnAmmoForWeapon` · `RemoveAmmoForWeapon`

**Contract** — multiplayer convenience: a weapon flagged for automatic ammunition gets a
full complement created into the actor's inventory when it is acquired, and that
ammunition destroyed when the weapon leaves. Both are authoritative-side only.

**Notes** — the removal destroys the ammunition unconditionally, without checking whether
another carried weapon uses the same type. The check exists in the source, commented out.
Reproducing the shipped behaviour means destroying it; a rebuild should probably restore
the check and note the divergence.
