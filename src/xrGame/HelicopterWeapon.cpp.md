# src/xrGame/HelicopterWeapon.cpp

> The helicopter's guns: a turret aimed within the model's own joint limits, a cannon whose burst pattern is a line of fire walked across the target, and a pair of rocket pods fired in a configured cadence.

**Needs** — [`helicopter.h`](helicopter.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`ShootingObject.h`](ShootingObject.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`HudSound.h`](HudSound.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`Level.h`](Level.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`Helicopter.cpp`](Helicopter.cpp.md)
**Tier floor** — T2: angle arithmetic in a bone's bind space and a per-shot cadence

## Purpose

Everything the helicopter does with its weapons. Two of the decisions here are unusual
enough to be the reason the file is worth reading.

**The turret is aimed in the model's bind space.** The aim is not "point this bone at that
world position"; it is "what two angles, relative to the rest pose, would make the
barrel point there", clamped to the joint limits the *model* declares. If the clamp bites,
the target is out of the turret's envelope and firing is refused. A rebuild that aims the
bone directly will happily shoot through the airframe.

**The cannon does not shoot at the target.** With the fire-trail option on, it shoots at a
point that sweeps along a line *through* the target — in from one side, out the other —
so that the impacts walk across the ground toward the player and past them. That is the
signature behaviour of this encounter: the player sees dust kicking toward them and has
time to run. A rebuild that aims at the target produces an instant kill and a different
game.

## Turret aiming — `UpdateMGunDir`

**Contract** — recompute the fire point and direction from the fire bone, the two rocket
launch transforms, the two turret target angles, and whether firing is permitted.
Permitted only when both angles are inside the joint limits *and* the turret has actually
swung to within the configured tolerance of them.

**Invariants** — three separate conditions can forbid firing and all three must hold:
elevation within limits, traverse within limits, and the current turret angles close
enough to the target angles. The third is what stops the gun firing while it is still
slewing.

```text
FUNCTION update_turret_aim()
  fire_xform = fire bone's transform composed with the helicopter's
  fire_point = fire_xform origin
  fire_dir   = normalize(enemy position - fire_point)
  left_launch, right_launch = the two rocket bones' transforms, composed and raised

  allow_fire = true
  local_target = enemy position expressed in the helicopter's own space
  FOR each of the two turret axes (elevation, traverse)
    offset  = local_target - that bone's bind origin
    in_bind = offset rotated into that bone's inverse bind orientation, normalized
    target_angle = normalize_signed(bind angle - in_bind's angle about this axis)
    clamped = clamp(target_angle, the model's joint limits for this axis)
    IF clamped differs from target_angle THEN allow_fire = false
    target_angle = clamped
  IF either axis's current angle differs from its target by more than the
     configured barrel tolerance THEN allow_fire = false
```

**Notes** — the two rocket launch points are raised a metre above their bones, with the
source marking both as fake. The rockets are therefore launched from above the pods,
which keeps them clear of the airframe's own collision geometry on the first frame of
flight. It is a workaround for self-collision, not a placement decision.

The elevation limit is read from the first of the model's joint limit triples and the
traverse from the second, matching how the two bones are authored. Both clamps are
applied *negated* — the model's limits are expressed in the opposite sense from the
angles computed here.

## `BoneMGunCallbackX` / `BoneMGunCallbackY`

**Contract** — per-bone transform callbacks installed on the two turret bones at spawn.
Each post-multiplies its bone's animated transform by a rotation of the current turret
angle about its axis. Called by the animation system every time the pose is evaluated.

**Invariants** — the callbacks compose *onto* the animated pose rather than replacing it,
so an idle animation on the turret still plays underneath the aim. This is the standard
way the engine layers procedural aim over authored animation, and the two angles are the
only state the callbacks read.

## `UpdateWeapons`

**Contract** — the per-frame weapons tick. Aims the turret when engaged and returns it to
rest when not; slews the turret's current angles toward the target angles at a fixed rate;
and, when engaged and permitted, starts the cannon if the target is within the cannon's
distance band and launches rockets when the target is within the rocket band and the
cadence has elapsed. Disengaging or being forbidden to fire stops the cannon.

**Invariants** — the cannon's and the rockets' distance bands are separate and may not
overlap; the distance used is **horizontal**, ignoring altitude, which is what makes the
engagement envelope a ring on the ground rather than a sphere.

```text
FUNCTION update_weapons()
  IF engaged THEN update_turret_aim() ELSE target angles = rest
  slew current turret angles toward the target angles at a fixed rate
  IF engaged AND firing is permitted THEN
    d = horizontal distance to the target
    IF d is inside the cannon band THEN cannon_start()
    IF d is inside the rocket band AND the rocket cadence has elapsed THEN
      IF rockets are synchronized THEN launch both pods
      ELSE                             launch the pod that did not fire last
      record the launch time
  ELSE
    cannon_stop()
  cannon_tick()
```

## `MGunFireStart`

**Contract** — begin a cannon burst. Refused outright when the configuration disables the
cannon. When the fire trail is enabled and a burst is not already running, records the
burst's start time and computes the trail's length for *this* burst.

**Invariants** — the trail length is derived from geometry, not configured directly: the
helicopter's height above the target and the turret's maximum depression give the nearest
ground point the gun can reach; if the target is further away horizontally than that, the
shortfall doubled is the trail length, capped at the configured desired length. A target
directly below, or above, gets the full configured length.

```text
FUNCTION cannon_start()
  IF the cannon is disabled THEN RETURN
  IF a burst is already running OR the trail is off THEN just start firing
  height = fire point altitude - target altitude
  IF height > 0 THEN
    reach = height * tan(maximum depression)      # nearest ground point the gun can hit
    horizontal = horizontal distance to the target
    IF horizontal > reach THEN
      trail_length = min(2 * (horizontal - reach), desired trail length)
    ELSE trail_length = desired trail length
  ELSE trail_length = desired trail length
  burst_start_time = now
  start firing
```

**Notes** — the derivation means a helicopter that is *too high and too far* shortens its
own walking line so that the walk still ends on the target rather than stopping short
where the turret runs out of depression. It is a correction for the turret's envelope
expressed as a change to the burst pattern, which is an unusual and rather elegant move.

## `OnShot` — one round

**Contract** — fire a single round. With the trail off, straight down the aim direction.
With the trail on, at a point on the line through the target: the walk position is
derived from the elapsed burst time at a fixed trail speed, mirrored around the midpoint
so the line sweeps in and back out, and scattered by a configured trace width. When the
walk runs past the end of the line, the burst ends. Emits the muzzle particles, the
muzzle light, the smoke, the ejected casing and the shot sound.

**Invariants** — the walk is measured from the burst's start time, so a burst is a
*function of time*, not of shot count. Frame rate does not change where the impacts land.

```text
FUNCTION on_shot()
  origin = current fire point ; direction = current aim direction
  IF the trail is on THEN
    elapsed = now - burst start
    remaining = trail_length - elapsed * trail_speed
    IF remaining < 0 THEN cannon_stop() ; RETURN         # the line is walked out
    # a level line from the muzzle's horizontal position through the target
    along = normalize(target - (origin at the target's altitude))
    IF remaining is in the first half THEN
      walk outward: flip `along`, offset = remaining - half
    ELSE
      walk inward:  offset = half - remaining
    aim_point = target + along * offset + a random scatter within the trace width
    direction = normalize(aim_point - origin)
  fire one round from origin along direction, with the base dispersion and the loaded
    cartridge, attributed to this helicopter
  muzzle particles, muzzle light, smoke, casing ejection, shot sound
```

**Notes** — the trail speed is clamped between the helicopter's own current speed and a
large ceiling, starting from a hard-coded fifteen. The lower clamp is the interesting
part: the line of fire must sweep at least as fast as the machine is flying, or the walk
would fall behind the helicopter and the impacts would trail rather than lead.

The per-shot random seed is drawn from a small range and passed into the bullet, which is
how the shot's dispersion is made reproducible across the network.

## `MGunUpdateFire`

**Contract** — the cannon's cadence. Counts the shot timer down by frame time, keeps the
muzzle particles and light alive, and fires a round each time the timer reaches zero,
adding one shot interval back. Additionally implements an optional **duty cycle**: when
both a fire time and a no-fire time are configured, the cannon alternates between firing
for one and holding for the other.

**Invariants** — the timer accumulates rather than resets, so a frame long enough to span
two shot intervals still fires both rounds' worth of cadence rather than losing one. When
not firing, the timer is clamped non-negative so that resuming does not discharge a
backlog.

**Notes** — the two duty-cycle keys are read from configuration *inside the per-frame
update*, on every frame. That is a performance wart, not a decision; a rebuild reads them
at load.

## `startRocket`

**Contract** — launch one rocket from the named pod, if any remain loaded and rockets are
enabled. Attributes it to the helicopter, builds a launch frame pointing at the target
from that pod's raised transform, gives it the launcher's configured speed, announces the
launch as a network event, drops it from the loaded set, records which pod fired, and
plays the launch sound.

**Invariants** — the launch orientation is a full orthonormal frame built from the aim
direction, not just a direction, because the rocket is a physical object that needs an
up axis. The network announcement carries the rocket's own entity identifier so every
client detaches the same one.

## `OnEvent`

**Contract** — three entity events. Taking ownership of a rocket attaches it to a pod;
rejecting ownership or launching detaches it, with the launch flag distinguishing "it
flew away" from "it was removed".

**Invariants** — rockets are real entities with their own identifiers, spawned and
attached rather than conjured at launch. That is why the scheduled tick in
[`Helicopter.cpp`](Helicopter.cpp.md) tops the loaded count back up to four: the pods hold
actual objects.

## `MGunFireEnd`, `get_ParticlesXFORM`, `get_CurrentFirePoint`

**Contract** — stop firing, stop the muzzle flame and clear the burst's start time; and
hand out the fire bone's world transform and the fire point, which the shooting base needs
for its particle placement.
