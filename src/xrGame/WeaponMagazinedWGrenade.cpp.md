# src/xrGame/WeaponMagazinedWGrenade.cpp

> A rifle with an under-barrel grenade launcher: two complete weapons in one object, switched by swapping every ammunition field between a primary set and a secondary set.

**Needs** — [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md) · [`GrenadeLauncher.h`](GrenadeLauncher.h.md) · [`ExplosiveRocket.h`](ExplosiveRocket.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`player_hud.h`](player_hud.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a ballistic solve per grenade, and a per-frame animation selection with eight branches.

## Purpose

The engine's weapon model assumes one weapon has one magazine, one ammunition type list
and one set of fire parameters. An under-barrel launcher breaks all three. Rather than
generalize the model, this file keeps a **shadow copy** of every ammunition field and
swaps the two sets when the player toggles modes.

That decision is the whole file. It means every inherited procedure works unchanged in
either mode — the burst loop, the reload, the heads-up readout all read the same fields
they always did — at the cost of every *other* procedure having to ask which mode is
active, and of the shadow set being invisible to anything that does not know about it.

A rebuild should model a weapon as *a list of firing assemblies* with one selected. The
swap is a workaround for not having done that.

## State

```text
RECORD GrenadeRifle EXTENDS MagazinedWeapon, RocketLauncher
  grenade_mode      : bool              # which set is currently the primary
  ammo_types_2      : list<text>        # the shadow ammunition type list
  ammo_type_2       : int (8-bit)
  magazine_2        : list<Cartridge>   # the shadow magazine
  magazine_size_2   : int               # the RIFLE's capacity, parked while in
                                        # grenade mode (the launcher's is always 1)
  default_cartridge_2 : Cartridge
  ammo_elapsed_2    : int (8-bit)
  launch_speed      : real              # the grenade's muzzle velocity
```

**Invariants** —

1. The *active* set always lives in the inherited fields; the *inactive* set lives in the
   shadow fields. Nothing else in the hierarchy knows the shadow set exists.
2. In grenade mode the magazine capacity is 1. In rifle mode it is `magazine_size_2`,
   which therefore holds the rifle's capacity at all times — the naming is inverted from
   what the suffix suggests, and this trips every reader.
3. A live grenade object is attached whenever the launcher's magazine is non-empty, in
   exactly the same way as in [`WeaponRPG7.cpp`](WeaponRPG7.cpp.md). The `grenade` bone's
   visibility on the first-person model tracks "loaded, or reloading".
4. `ammo_elapsed_2` is written at spawn and never read again; the shadow magazine's
   length is the real figure.

## `PerformSwitchGL` — the swap

**Contract** — exchanges every ammunition field between the active and shadow sets. Pure
bookkeeping; no animation, no sound, no validation.

```text
FUNCTION perform_switch(rifle)
  rifle.grenade_mode = NOT rifle.grenade_mode
  rifle.magazine_size = 1 IF grenade_mode ELSE rifle.magazine_size_2
  swap rifle.ammo_types        WITH rifle.ammo_types_2
  swap rifle.ammo_type         WITH rifle.ammo_type_2
  swap rifle.default_cartridge WITH rifle.default_cartridge_2
  swap rifle.magazine          WITH rifle.magazine_2
  rifle.ammo_elapsed = length of rifle.magazine
  invalidate the heads-up ammunition cache
```

**Invariants** — the swap must be *complete*. Missing any one field leaves the weapon
firing rifle rounds from the launcher's magazine or reporting the wrong reserve. The set
is: type list, selected type, prototype cartridge, magazine, and the derived count and
capacity.

## `SwitchMode` — the user-visible toggle

**Contract** — refuses unless the weapon is in a settled state (idle, hidden, jammed or
magazine-empty), is not pending, and actually has a launcher attached. On success it
leaves zoom, marks the weapon pending, performs the swap, plays the switch sound and
animation, and invalidates the ammunition cache.

```text
FUNCTION switch_mode(rifle) -> bool
  RETURN false unless can_switch(rifle)
  leave zoom
  rifle.pending = true
  perform_switch(rifle)
  play the switch sound at the muzzle
  play the mode-switch animation
  invalidate the heads-up ammunition cache
  RETURN true

FUNCTION can_switch(rifle) -> bool
  RETURN (state is idle, hidden, misfire or magazine-empty)
     AND NOT rifle.pending
     AND a grenade launcher is attached
```

**Invariants** — the swap happens *before* the animation plays, so the animation selector
(which branches on `grenade_mode`) already sees the new mode and plays the correct
direction's clip. The state machine reaches a dedicated *switch* state whose animation
end returns the weapon to idle.

**Notes** — the mode-switch animation probes two names per direction and, if neither
exists, re-enters the switch state immediately — which the state table turns into an
instant return to idle. So a weapon with no switch animation toggles instantly rather
than failing.

## Firing a grenade

### `Action` — the launcher's trigger

**Contract** — in grenade mode the fire binding is intercepted entirely, before the
inherited handler sees it. There is no burst, no shot clock and no per-frame firing
state: the grenade leaves on the key-down.

```text
IF grenade_mode AND the command is fire THEN
  RETURN false IF pending
  IF key down THEN
    IF the magazine is non-empty THEN launch_grenade(rifle) ELSE reload
    IF the state is idle THEN play the empty click
  consume the command
```

The mode-switch binding is a separate function key, and it enters the switch state rather
than calling the toggle directly, so the transition goes through the state machine.

### `LaunchGrenade`

**Contract** — identical in shape to the revolving launcher's launch (see
[`WeaponRG6.cpp`](WeaponRG6.cpp.md)), with the same single-player scoped aim assist.

```text
FUNCTION launch_grenade(rifle)
  RETURN IF no grenade object is attached
  position = rifle.second_muzzle           # the launcher's own fire point
  ask the carrier for its aim direction
  IF single player THEN position = rifle.second_muzzle    # re-assert; the carrier's
                                                          # aim gives the rifle's muzzle
  launch_frame = a basis with forward = direction, origin = position

  IF single player AND scoped AND the carrier is the actor THEN
    disable collision on the carrier and the weapon
    hit = ray query along direction, 300 m, static geometry only
    re-enable collision
    IF hit THEN
      solutions = throw directions carrying (direction * hit.range) at launch_speed
                  under the effective gravity
      IF any exists THEN direction = the flatter one

  launch the attached grenade from launch_frame at direction * launch_speed
  record the carrier as its initiator
  IF locally owned AND authoritative THEN
    pop one round; ammo_elapsed = ammo_elapsed - 1
    broadcast a launch-rocket event naming the grenade
```

**Invariants** — the round is consumed only on the authoritative side, and the grenade is
detached by the returning event rather than here, so both sides agree on which object
left and when.

The **second** fire point is used, not the first: the launcher's muzzle sits below the
rifle's, and the grenade must come out of it. This is the only place the second fire
point matters for a rifle.

### `OnEvent` — the launch's effects

**Contract** — the three rocket ownership events. On a *launch* event the shot's visible
and audible effects are played here rather than at the launch site — the animation, the
launcher's own shot sound at the second muzzle, the camera recoil, and the second
muzzle's flame particles. Playing them on the event rather than on the launch is what
makes a remote player's launcher flash at the same moment on every machine.

## Two-mode behaviour

Every inherited entry point that behaves differently in grenade mode branches on the
flag. The branches, and the reason each exists:

| Entry point | In grenade mode |
|---|---|
| `OnShot` | play the launcher's animation, sound at the second muzzle, recoil and second-barrel particles — instead of the rifle's |
| `switch2_Reload` | play the launcher's reload sound at the second muzzle and its own reload animation |
| `state_Fire` | **does nothing** — the grenade already left on the key-down |
| `FireEnd` | go to the weapon base's, skipping the magazined weapon's auto-reload |
| `OnMagazineEmpty` | click if idle, and nothing else — no magazine-empty state, no auto-reload |
| `UseScopeTexture` | false — a scope picture is not drawn over a grenade sight |
| `CurrentZoomFactor` | the iron-sight factor, even with a scope attached — the launcher has its own sight |
| `GetCurrentHudOffsetIdx` | 2 while aiming, versus 1 for the rifle — a third authored hand pose |
| `OnAnimationEnd` on fire | reload automatically; a one-round launcher is always empty after firing |

**Invariants** — the disabled per-frame firing state is the load-bearing one. The
commented-out body in the source shows the burst loop it would have been; leaving it
empty is what makes the launcher a single-shot, immediate weapon.

## `ReloadMagazine` · `net_Spawn`

**Contract** — both must bring the grenade *object* into step with whichever magazine is
the launcher's.

The spawn path is the awkward one, because the mode the weapon spawns in is not known
until the record is applied, and multiplayer and single player handle it differently:

```text
FUNCTION spawn_from_record(rifle, record)
  base.spawn_from_record(record)
  set the first-person grenade bone's visibility from the rifle's loaded count
  clear pending
  ammo_elapsed_2 = record.grenade_count
  ammo_type_2    = record.grenade_type
  default_cartridge_2 = load(ammo_types_2[ammo_type_2])

  IF multiplayer THEN
    IF in rifle mode AND a launcher is attached AND no grenade is attached
       AND the record said grenades were loaded THEN
      push one round onto the shadow magazine
      spawn the grenade object named by that round's "fake_grenade_name"
  ELSE
    # single player: the launcher's magazine is whichever one is NOT active
    IF (grenade mode AND the active magazine is non-empty AND no grenade attached)
       OR (rifle mode AND the shadow magazine is non-empty AND no grenade attached) THEN
      spawn the grenade object named by that magazine's top round's "fake_grenade_name"
```

**Notes** — in multiplayer exactly one grenade is ever restored, regardless of the
record's count, and the shadow magazine gets one round rather than the recorded number.
The launcher holds one round, so this is consistent, and it is why the record's count is
otherwise unused.

The grenade's class comes from the *ammunition section's* `fake_grenade_name`, the same
indirection the other launchers use: one round section describes both the carried item
and the flying object.

## Attaching the launcher

**Contract** — attaching or detaching a grenade launcher is handled here rather than by
the magazined weapon, because it carries the launch speed and because detaching must
empty the launcher's magazine.

```text
FUNCTION detach(rifle, section, spawn_item) -> bool
  IF this is our launcher AND it is attached THEN
    clear the launcher bit
    IF NOT grenade_mode THEN perform_switch(rifle)   # make the launcher's set active
    unload_magazine(rifle)                            # its grenades go back to inventory
    perform_switch(rifle)                             # restore the previous mode
    refresh addon visibility
    IF idle THEN replay the idle animation
    RETURN the base item's detach
  RETURN base.detach(section, spawn_item)
```

**Invariants** — the swap-unload-swap dance is the only way to reach the shadow magazine
with the inherited unload, which operates on the active one. It leaves `grenade_mode`
where it started, which matters because the player may have been in rifle mode.

Attaching reads the launch speed from the *attached launcher item* rather than from the
weapon's section, so two launchers of different power can fit the same rifle. When the
launcher is permanent instead, the speed comes from the weapon's own section; when
attachable, `InitAddons` re-reads it from the launcher's section after any addon change.

## Animations — the eight-way selector

**Contract** — every animation slot has up to three variants: *no launcher*, *launcher
attached but in rifle mode* (`_w_gl`), and *grenade mode* (`_g`). Each variant probes a
newer and an older name.

The idle selector is the largest, because it crosses the mode with the actor's movement
state:

```text
FUNCTION play_idle(rifle)
  IF no launcher is attached THEN base.play_idle(); RETURN
  IF zoomed THEN
    play the aim idle for the current mode
    RETURN
  movement = sprint IF the actor is sprinting
             ELSE crouch IF the actor is moving and crouched
             ELSE moving IF the actor is moving
             ELSE idle
  play the (mode, movement) idle
  # the crouch variant is optional: when absent, fall through to the moving one
```

**Invariants** — the crouch-while-moving variant is probed and falls through to the plain
moving variant, so animation sets without it still work. The aim idle in grenade mode is
played with blending enabled where the rifle's is not, to avoid a visible snap when
switching modes while aiming — a fix noted in the source.

## Persistence and replication

**Contract** — the save payload adds, after the magazined weapon's: the mode flag and the
shadow magazine's length. On load, if the saved mode differs from the current one the
weapon **switches modes**, which runs the full swap and plays the switch animation; then
the shadow magazine is refilled to the saved length with the shadow type's rounds.

The network payload puts the mode flag **first**, before the inherited payload, and the
import switches modes before reading the rest. That ordering is required: the inherited
import writes the ammunition count into the *active* magazine, so the mode must already
be right.

**Invariants** — switching modes during a load or a network import also plays a sound and
an animation, which is audible on a save load. A rebuild should separate the swap from
its presentation.

The shadow magazine is restored as homogeneous rounds of the shadow type, so — as with
every other weapon — a mixed launcher magazine does not survive. A one-round launcher
makes this moot.

## `GetBriefInfo`

**Contract** — the heads-up readout, with the per-type counts drawn from whichever type
list is active, plus one extra field: the **grenade count**, always showing the
*launcher's* reserve regardless of the current mode, as a number, a dash when ammunition
is unlimited, or an `X` when there are none. The field is cleared and the call reports
"nothing changed" when no launcher is attached.

**Notes** — the total-reserve figure here is the suitable total minus the magazine, while
the magazined weapon's version sums the first three per-type counts. The two disagree for
a weapon with more than three ammunition types.

## `Weight` · `IsNecessaryItem` · upgrades

**Contract** — weight adds the shadow magazine's rounds to the inherited figure.
`IsNecessaryItem` accepts a section from *either* type list, so a stalker picks up
grenades for a rifle he is carrying in rifle mode.

The two upgrade installers are the one place the shadow set is addressed by name rather
than by swapping, and they do it inconsistently: the ammunition-class installer writes
`ammo_types_2` when in grenade mode and `ammo_types` otherwise, while the grenade-class
installer writes the opposite. Read together they are correct — each writes the list that
is currently *not* the launcher's, respectively *is* — but the pair is impossible to read
without tracing the swap. Both reset the selected type to zero in both sets. The
ammunition-class installer also writes the rifle capacity into `magazine_size_2` and then
re-derives the active capacity from the mode.
