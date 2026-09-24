# src/xrGame/WeaponRPG7.cpp

> The rocket launcher: a single-shot weapon whose loaded round is a visible rocket on the model and a real object in the world at the same time.

**Needs** — [`WeaponRPG7.h`](WeaponRPG7.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`player_hud.h`](player_hud.h.md) · [`Level.h`](Level.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: bone visibility and a launch transform per shot.

## Purpose

Two problems that the ordinary weapon path does not have.

First, the **loaded round is visible**. The launcher's model has a rocket bone that must
appear exactly when a rocket is loaded, on both the world model and the first-person
model — and, during a reload, on the first-person model slightly earlier than on the
world model, because the reload animation shows the rocket being slid in. Every path
that can change the loaded count therefore refreshes that visibility, which is why the
file is mostly one procedure called from nine places.

Second, like the revolving launcher, it fires an **object** rather than a bullet: a real
rocket with its own physics, fuse and explosion, spawned when the weapon is loaded and
detached when it is launched.

## State

```text
RECORD RocketLauncherWeapon EXTENDS CustomPistol, RocketLauncher
  rocket_section : text     # the projectile class to spawn, from "rocket_class"
```

**Invariants** —
- a rocket object is attached whenever the magazine is non-empty;
- the `grenade` bone's visibility on the world model tracks the loaded count exactly;
- on the first-person model it tracks *loaded count OR currently reloading*.

## `UpdateMissileVisibility` — the file's centre

**Contract** — recomputes the two visibilities from the loaded count and the state. Cheap
and idempotent, which is why it is simply called after anything that could matter.

```text
FUNCTION update_missile_visibility(launcher)
  hud_visible   = (ammo_elapsed > 0) OR (state = reload)
  world_visible = (ammo_elapsed > 0)
  IF the first-person model is active THEN
    set the "grenade" bone's visibility on it TO hud_visible
  set the "grenade" bone's visibility on the world model TO world_visible
```

**Invariants** — the asymmetry is the point. During the reload animation the player must
see the rocket in his hands going in, but a bystander must not see it appear on the tube
before it is seated. The magazine transfer happens at the animation's end, so the world
model turns it on then and the first-person model has had it on since the animation
started.

It is called from: spawn, every state transition, network import, the shot's trace, the
reload, the unload, a launch event, and the moment the first-person model is attached.
That last one is what stops a redrawn launcher showing the wrong state for a frame.

## `switch2_Fire` — the launch

**Contract** — entering the fire state arms one shot and, if a rocket is attached,
launches it immediately. There is no burst loop and no shot clock involvement: the whole
shot happens in this one procedure.

```text
FUNCTION switch_to_fire(launcher)
  launcher.shots_fired      = 0
  launcher.fire_single_shot = true
  launcher.firing           = false

  RETURN unless the state is fire AND a rocket is attached

  position, direction = launcher.muzzle, launcher.aim_direction
  IF the carrier is an entity THEN
    ask the carrier for its aim: position, direction = carrier.fire_params(launcher)
    IF the first-person model is active THEN
      # aim at what the crosshair's ray actually hit, not along the sight line:
      # the muzzle is offset from the eye, so the two diverge at close range
      target = eye position + carrier's aim direction * the crosshair ray's range
      position  = launcher.muzzle
      direction = normalize(target - launcher.muzzle)

  launch_frame = a basis with forward = direction, origin = position
  velocity     = normalize(direction) * launcher.muzzle_speed

  launch the attached rocket from launch_frame with that velocity
  record the carrier as the rocket's initiator            # for kill attribution
  IF we are authoritative THEN broadcast a launch-rocket event naming the rocket
```

**Invariants** — the parallax correction is the interesting decision, and it is the same
problem the fire dependencies solve for bullets: the projectile must appear to come from
the crosshair. Here it is solved by re-aiming from the muzzle at the point the crosshair's
ray hit, so a rocket fired at a nearby wall lands on the crosshair rather than a metre to
the side.

The rocket is **not** detached here — unlike the revolving launcher, which detaches
immediately. Detachment happens when the launch event comes back through `OnEvent`, so
the two sides agree on which object left. The cost is one round trip during which the
launcher still believes it holds a rocket that is already flying.

## `FireTrace`

**Contract** — runs the ordinary shot (consuming the round, wearing the weapon, emitting
particles) and then refreshes the rocket's visibility, which the now-empty magazine has
changed.

**Notes** — the ordinary shot also creates a *bullet*. A rocket launcher therefore fires
both a traced bullet and a physical rocket on the same trigger pull; the bullet's
parameters in the shipped data make it harmless. A rebuild should suppress the bullet.

## `net_Spawn` · `ReloadMagazine` · `UnloadMagazine`

**Contract** — three places where the rocket object must be brought into step with the
magazine:

- **spawn** — refresh visibility, then, if the record said a round was loaded and no
  rocket is attached, spawn one of the configured class;
- **reload** — after the ordinary transfer, if the magazine is non-empty and no rocket is
  attached, spawn one;
- **unload** — after the ordinary transfer, refresh visibility. Note it does **not**
  destroy the attached rocket, so a launcher unloaded through the inventory keeps a
  rocket object attached to an empty tube until the next reload's guard notices.

## `OnEvent` — rocket ownership

**Contract** — the same three events the revolving launcher handles: ownership taken
attaches, ownership rejected detaches without launching, and launched detaches as
launched — the last additionally refreshing visibility.

## `Load`

**Contract** — loads both halves, then overrides the scope zoom factor from the section's
`max_zoom_factor` and reads the rocket class name. The zoom override exists because a
launcher's optic is a fixed magnifier authored under a different key than every other
weapon's — one more frozen name.

## `AllowBore`

**Contract** — the idle fidget is suppressed on an empty launcher: the animation shows
the rocket being inspected, and there is nothing there.

## `PlayAnimReload`

**Contract** — plays the reload animation without mixing into the current pose. Every
other weapon's reload blends in; a launcher's must start from the exact rest pose because
the rocket bone's appearance is keyed to it.

## Pass-throughs

`SwitchState` and `FireStart` forward unchanged and should be deleted.
