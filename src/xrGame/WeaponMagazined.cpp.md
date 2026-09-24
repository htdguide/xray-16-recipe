# src/xrGame/WeaponMagazined.cpp

> The firing state machine every conventional firearm runs: draw, idle, burst, jam, reload, holster — plus the addon attachment rules, the fire-mode selector and the ammunition accounting that reload performs.

**Needs** — [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Weapon.h`](Weapon.h.md) · [`Scope.h`](Scope.h.md) · [`Silencer.h`](Silencer.h.md) · [`GrenadeLauncher.h`](GrenadeLauncher.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Actor.h`](Actor.h.md) · [`EffectorZoomInertion.h`](EffectorZoomInertion.h.md) · [`HudSound.h`](HudSound.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a state machine over a real-valued shot clock, running every frame for the held weapon.

## Purpose

This is where a weapon's *feel* lives. Everything above it decides where the muzzle is
and how wide the cone is; this file decides what happens when the trigger goes down: how
soon the first round leaves, at what rate the rest follow, when the burst stops, what the
animation and the sound do, and what the player sees when it jams.

It is also the addon authority. `CWeapon` knows *whether* an addon is attached; this
file knows what attaching one costs, which items are acceptable, and which parameters
are re-read as a result.

Almost every firearm in the game is this class or a subclass of it. The exceptions are
the knife, the static machine gun and the binoculars.

## State

```text
RECORD MagazinedWeapon EXTENDS Weapon

  # --- burst control ----------------------------------------------------
  queue_size        : int      # rounds this pull of the trigger may fire;
                               # negative means "until the magazine is empty"
  shots_fired       : int      # rounds fired since the burst began
  stopped_after_queue : bool   # the burst ended because it reached queue_size
  fire_single_shot  : bool     # guarantee at least one round even on a tap

  # --- fire modes -------------------------------------------------------
  has_fire_modes    : bool
  fire_modes        : list<int>   # authored, e.g. [1, 2, -1]; -1 is full auto
  current_fire_mode : int         # index into fire_modes
  preferred_fire_mode : int       # authored; unused today

  # --- the Abakan rule (see below) --------------------------------------
  base_dispersioned_bullets_count : int   # first N rounds of a burst are special
  base_dispersioned_bullets_speed : real  # and leave at this muzzle velocity
  old_bullet_speed  : real                # the normal velocity, saved across the burst
  burst_origin_position, burst_origin_direction : vector3

  # --- reload -----------------------------------------------------------
  lock_ammo_type    : bool     # a recursive reload pass must not change type
  current_ammo_box  : optional<AmmoBox>   # the box being drawn from

  # --- sound ------------------------------------------------------------
  current_shot_sound : text    # "sndShot" or "sndSilencerShot"
  silencer_flame_particles, silencer_smoke_particles : optional<text>
  sounds_enabled    : bool     # the carrier may forbid draw/holster/reload sounds
  layered_shot_sounds : SoundCollection   # several shot samples, played together
```

**Invariants** —

1. `shots_fired` is reset to zero on entering *idle* and on entering *fire*, and nowhere
   else. Everything that reads it — the Abakan rule, the wear figure, the burst stop —
   depends on those being the only two reset points.
2. `fire_single_shot` is set on entering *fire* and cleared by the first round of the
   burst. It is what makes a tap of the trigger fire exactly one round even when the
   release arrives before the next update.
3. `queue_size` negative means unlimited; it is never zero.
4. While `base_dispersioned_bullets_count` is non-zero, the muzzle velocity is
   *temporarily* replaced and the original is held in `old_bullet_speed`, which the idle
   transition restores. A burst that ends abnormally must still pass through idle or the
   weapon keeps the special velocity.

## The shot clock

There is one real-valued countdown, inherited from the shooting object, and everything
about rate of fire is expressed through it.

- `fShotTimeCounter` — seconds until the next round may leave.
- `fOneShotTime` — the normal interval between rounds (rounds per minute, inverted).
- `fModeShotTime` — an alternative interval used for two-round bursts and for the first
  rounds of a cycle-down weapon.

In every state that is *not* firing, the counter simply decays with frame time and is
clamped at zero, so a weapon that was mid-cadence when it was holstered is ready
immediately on redraw rather than carrying a stale debt.

## `state_Fire` — the burst loop

**Contract** — the core of the file. Runs once per frame while the weapon is settled in
the fire state, and may emit several rounds in one call when the frame was long.

```text
FUNCTION state_fire(weapon, frame_time)
  IF weapon.ammo_elapsed > 0 THEN
    position, direction = weapon.muzzle, weapon.aim_direction
    RETURN IF there is no carrier
    RETURN IF the carrier is a corpse-loot container        # a known bad state
    IF the carrier's active item is nothing THEN
      log the inconsistency; stop shooting; RETURN          # guards a load-time crash

    ask the carrier for its aim: position, direction = carrier.fire_params(weapon)
    IF the carrier is no longer holding the trigger THEN stop shooting

    IF weapon.shots_fired = 0 THEN
      weapon.burst_origin_position  = position
      weapon.burst_origin_direction = direction

    WHILE the magazine is non-empty
      AND the shot clock has expired
      AND (the weapon is firing OR fire_single_shot is set)
      AND (queue_size < 0 OR shots_fired < queue_size)

      IF check_for_misfire(weapon) THEN stop shooting; RETURN

      weapon.fire_single_shot = false

      # rearm the clock: a two-round burst mode, or the early rounds of a
      # cycle-down weapon, use the alternative interval
      IF current fire mode = 2 OR (weapon cycles down AND shots_fired <= 1) THEN
        shot clock = fModeShotTime
      ELSE
        shot clock = fOneShotTime

      weapon.shots_fired = weapon.shots_fired + 1
      on_shot(weapon)                       # sound, camera, animation, shell, particles

      IF weapon.shots_fired > base_dispersioned_bullets_count THEN
        fire_trace(weapon, position, direction)              # live aim
      ELSE
        fire_trace(weapon, burst_origin_position, burst_origin_direction)

    IF weapon.shots_fired = queue_size THEN weapon.stopped_after_queue = true
    update sound positions

  IF the shot clock has expired THEN
    IF the magazine is empty THEN on_magazine_empty(weapon)
    IF the shooting animation has finished THEN stop shooting
  ELSE
    shot clock = shot clock - frame_time
```

**Invariants and why they matter** —

- The loop is a `WHILE`, not an `IF`. At low frame rates a fast weapon must still emit
  the right number of rounds; a per-frame cap would make rate of fire depend on
  performance.
- The misfire roll happens **before** the round is consumed and before the clock is
  rearmed, so a jam costs no ammunition and leaves the weapon immediately re-armable.
- The aim is re-read from the carrier every *frame*, not every round. Within one frame's
  worth of rounds the direction is fixed, which is invisible at any real frame rate and
  saves a query per round.
- The burst is only stopped once the shooting *animation* has finished, not when the
  clock runs out. That is why a single shot from a slow weapon holds the fire state for
  the length of its animation instead of snapping back to idle.

**The Abakan rule** — `base_dispersioned_bullets_count` exists for one weapon: a rifle
whose first two rounds of a burst leave at a higher velocity and from the *origin the
burst started at*, so that both rounds land in the same hole before recoil moves the
muzzle. The implementation is two coupled decisions: this loop chooses the frozen origin
for those rounds, and `FireBullet` swaps the muzzle velocity for them. It is the only
place in the engine where a weapon's rounds do not come from where the weapon currently
points, and it is the one piece of "weapon feel" that is genuinely special-cased.

## `FireBullet` — the velocity swap

**Contract** — wraps the base shot with the Abakan velocity substitution.

```text
FUNCTION fire_bullet(weapon, position, direction, dispersion, cartridge, ...)
  IF base_dispersioned_bullets_count > 0 THEN
    IF shots_fired <= 1 THEN
      weapon.old_bullet_speed = current muzzle velocity
      set muzzle velocity TO base_dispersioned_bullets_speed
    ELSE IF shots_fired > base_dispersioned_bullets_count THEN
      restore muzzle velocity TO weapon.old_bullet_speed
  base.fire_bullet(position, direction, dispersion, cartridge, ..., ammo_elapsed)
```

**Notes** — the save happens at `shots_fired <= 1`, which is *after* the counter was
incremented, so it runs on the first round. The restore is asymmetric (it runs on every
round past the special count, not just the first one past it), which is harmless but
redundant. The real safety net is the idle transition, which restores the velocity
unconditionally.

## `GetFireDispersion` — the honest crosshair

**Contract** — the special first rounds of a burst fire from the frozen origin, so their
effective cone is the *base* cone with no shooter contribution. But the crosshair must
not shrink while they are in flight, or the player learns to trust a cone that is about
to widen.

```text
FUNCTION fire_dispersion(weapon, cartridge_factor, for_crosshair) -> real
  d = base_dispersion(weapon, cartridge_factor)
  IF for_crosshair
     OR base_dispersioned_bullets_count = 0
     OR shots_fired = 0
     OR shots_fired > base_dispersioned_bullets_count THEN
    d = base.fire_dispersion(cartridge_factor)      # includes the shooter's own term
  RETURN d
```

## `OnStateSwitch` — the transition table

**Contract** — every state entry runs exactly one `switch2_*` procedure. The carrier's
permission to play draw, holster and reload sounds is sampled at the moment of entry,
not consulted later, so a carrier that goes quiet mid-reload still finishes its sound.

| Entering | Action |
|---|---|
| idle | reset the burst counter, restore the muzzle velocity, clear pending, play the idle animation |
| fire | validate that this is really the carrier's active item; arm the burst; on a client or a demo, start firing locally |
| misfire | if the actor is holding it, raise the "gun jammed" heads-up notice |
| magazine empty | leave zoom, try to reload; if that fails, play the empty click |
| reload | end firing, play the reload sound and animation, mark pending |
| showing | play the draw sound and animation, mark pending |
| hiding | leave zoom, end firing, play the holster sound and animation, mark pending — but only if we were not already hiding |
| hidden | end firing, cut the current animation without firing its end callback, signal hide complete, drop the recoil effector |

**Invariants** — *pending* is the "no input accepted" latch. It is set on entering
showing, hiding and reload and cleared on entering idle. Every input handler in the
hierarchy checks it first. Re-entering *hiding* is guarded because the holster can be
requested twice (by the inventory and by the network) and replaying the animation would
restart it.

The hidden transition cuts the animation *without* the end callback deliberately: the end
callback would switch the state again, and the weapon is already where it should be.

## `OnAnimationEnd` — the other half of the table

**Contract** — animation completion is what advances the machine out of every transient
state. This is the coupling that makes weapon timing follow the art rather than a
number.

```text
fire    -> idle
reload  -> perform the ammunition transfer, then idle
hiding  -> hidden
showing -> idle
idle    -> re-enter idle            # loops the idle animation
```

**Invariants** — **the magazine is refilled when the reload animation ends, not when it
starts.** Interrupting a reload therefore costs the player nothing and gains nothing,
which is the behaviour the games ship with.

## `FireStart` — the trigger goes down

```text
FUNCTION fire_start(weapon)
  IF weapon is jammed THEN
    notify scripts that the weapon jammed
    IF the actor holds it and is the view entity THEN raise the "gun jammed" notice
    play the empty click
    RETURN

  IF the magazine is empty THEN
    IF not already reloading THEN on_magazine_empty(weapon)
    RETURN

  IF already firing AND the weapon does not allow re-triggering while working THEN RETURN
  RETURN IF the state is reload, showing, hiding or misfire
  base.fire_start()
  IF ammo_elapsed = 0 THEN on_magazine_empty(weapon) ELSE switch to the fire state
```

The double empty check — once through the validity test and once after the base call —
is redundant but harmless.

## `FireEnd` — the trigger comes up

**Contract** — ends firing and, if the carrier is the actor and the magazine is now
empty and we are not already reloading, **reloads automatically**. Auto-reload is
therefore on trigger release, not on the last round — so a player who holds the trigger
through the end of a magazine reloads the instant they let go.

## `Reload` / `TryReload`

**Contract** — `TryReload` decides *whether* a reload is possible and switches to the
reload state if so; it does not move any ammunition. It also has an important side
effect: it notifies scripts of the available total, which is the hook mods use to grant
ammunition.

```text
FUNCTION try_reload(weapon) -> bool
  IF there is an inventory THEN
    IF single player AND the carrier is the actor THEN
      notify scripts: no-ammo-available, with the suitable total
    current_ammo_box = the inventory's first box of the current type

    IF the weapon is jammed AND has rounds THEN
      mark pending; switch to reload; RETURN true        # clearing a jam, not reloading

    IF a box was found, or ammunition is unlimited THEN
      mark pending; switch to reload; RETURN true

    FOR EACH type IN ammo_types
      box = the inventory's first box of that type
      IF box exists THEN
        current_ammo_box = box
        next_ammo_type_on_reload = that type's index
        mark pending; switch to reload; RETURN true

  IF the state is not idle THEN switch to idle
  RETURN false
```

**Invariants** — a jammed weapon with rounds still in it reloads to clear the jam, and
that path is checked *before* the ammunition search, so clearing a jam never requires
having a spare box.

## `ReloadMagazine` — the ammunition transfer

**Contract** — the only place rounds move from a box into a magazine. Runs when the
reload animation ends. Recursive, exactly once, to handle a partially filled box.

```text
FUNCTION reload_magazine(weapon)
  invalidate the ammunition-count cache
  clear the jam                                    # a reload always unjams
  IF NOT lock_ammo_type THEN forget the current box

  RETURN IF there is no inventory

  IF a pending ammo type was requested THEN
    adopt it; clear the request

  IF ammunition is not unlimited THEN
    RETURN IF the ammo type index is out of range
    current_ammo_box = the inventory's first box of the current type
    IF none AND NOT lock_ammo_type THEN
      scan every ammo type in order; adopt the first with a box present

  RETURN IF no box was found AND ammunition is not unlimited

  # a magazine may not hold two types at once
  IF NOT lock_ammo_type AND the magazine is non-empty
     AND the box's type differs from the magazine's THEN
    unload_magazine(weapon)                        # the old rounds go back to inventory

  IF the prototype cartridge is of the wrong type THEN reload it from the section
  WHILE ammo_elapsed < magazine_size
    IF ammunition is not unlimited THEN
      take one round from the box; BREAK if the box is empty
    ammo_elapsed = ammo_elapsed + 1
    push the round onto the magazine

  IF the box is now empty AND we are authoritative THEN mark it for disposal

  IF the magazine is still not full THEN
    lock_ammo_type = true
    reload_magazine(weapon)          # drain the next box of the SAME type
    lock_ammo_type = false
```

**Invariants** —

- the recursion depth is effectively one level per box consumed, and the type lock is
  what guarantees it terminates: the recursive pass may not switch types, so it either
  fills the magazine or finds no box and returns;
- a magazine is homogeneous **as a result of the unload-on-type-change rule**, not by
  construction — the data structure is a list of individually typed cartridges precisely
  so that the exceptions (a partially reloaded magazine in some mods) remain
  representable;
- `ammo_elapsed` and the magazine length stay equal at every one of the four assertion
  points in this procedure, which is the invariant that most of them exist to police.

## `UnloadMagazine`

**Contract** — empties the magazine back into the inventory, counting by type. Rounds go
into the first box of that type with room; whatever does not fit is spawned as new boxes
in the world, unless ammunition is unlimited. Notifies scripts that the magazine is
empty.

**Notes** — the per-type tally is built with a linear scan over an association list keyed
by raw string pointers. Comparing interned strings by pointer would have been exact and
constant-time; the scan compares content. Incidental.

## `OnMagazineEmpty`

**Contract** — notifies scripts with the remaining suitable total, then: if the weapon is
idle, just click; otherwise enter the magazine-empty state unless already heading for
that state or for a reload. The magazine-empty state exists only to hold the weapon for
one transition while it decides whether to auto-reload.

## `OnShot` — everything one round does besides flying

**Contract** — the fixed order of a shot's side effects. The order is load-bearing for
the particles: the particle frame must be forced to the true aim direction *before* the
smoke is emitted, or the smoke trails off in the model's forward direction.

```text
FUNCTION on_shot(weapon)
  play the current shot sound, layered, at the muzzle, routed as a HUD sound if the
      first-person model is active
  add the camera recoil
  play the shooting animation
  eject a shell at the ejection port, inheriting the weapon's linear velocity
  start the muzzle flame particles
  force the particle frame to the true aim direction
  start the smoke particles at the muzzle, inheriting the same velocity
```

**Notes** — "layered" shot sounds mean several samples are played at once (a close crack
and a distant report, say), which the sound collection resolves. The silenced variant is
a different entry in the same collection, selected once at addon time.

## Addons

### `CanAttach` / `CanDetach`

**Contract** — an item may be attached when it is the right kind (scope, silencer,
launcher), the corresponding slot is *attachable*, the slot is currently empty, and the
item's section is one this weapon accepts. For a scope, "accepts" means the item's
section appears as the `scope_name` of one of the weapon's candidate scope sections; for
the other two it is a single name match. Detaching mirrors the test with the slot
occupied.

**Notes** — the scope indirection (the weapon lists *scope sections*, each of which names
an *item section*) exists so one weapon can accept several different scopes with
different zoom factors and pictures. The other two addons never needed it.

### `Attach` / `Detach`

**Contract** — sets or clears the slot's bit, and for a scope also records which
candidate index was attached. On success the attached item is **destroyed**, not moved:
an attached addon has no object, only a bit and a section name. Both paths then refresh
addon visibility and re-run `InitAddons`, in that order.

**Invariants** — detaching a slot that is already empty is reported and treated as
success, because the network can deliver a detach twice.

The consequence of destroying the item is that a detached addon must be *spawned fresh*
from its section, which is why detach takes a "spawn the item" flag and why an addon
carries no condition of its own.

### `DetachScope`

**Contract** — finds which candidate scope section names the given item and resets the
current scope index to zero. Note it resets to zero rather than to "none": index zero is
the weapon's default optic, which for a weapon whose scope list is just itself is the
iron sight.

### `InitAddons` — re-reading the world after an addon change

**Contract** — recomputes every parameter that depends on which addons are attached.
Called after any attach, detach, spawn, network addon change or upgrade.

```text
FUNCTION init_addons(weapon)
  iron_sight_zoom_factor = section "ironsight_zoom_factor", default 50

  IF a scope is attached THEN
    IF the scope slot is attachable THEN
      read from the CURRENT SCOPE's section:
        picture, zoom factor, optional night-vision post-process,
        whether it zooms dynamically, optional living-target detector
      runtime zoom factor = the scope's zoom factor
      rebuild the scope overlay from the picture      # skipped on a dedicated server
  ELSE
    drop the scope overlay
    IF the weapon zooms at all THEN
      iron_sight_zoom_factor = section "scope_zoom_factor"

  silenced = a silencer is attached AND a silenced shot sound exists
  IF silenced THEN
    current flame and smoke particles = the silencer's
    current shot sound = the silenced one
    load the muzzle-flash light from the "silencer_" prefixed keys
  ELSE
    current flame and smoke particles = the weapon's own
    current shot sound = the plain one
    load the muzzle-flash light from the unprefixed keys

  IF a silencer is attached THEN apply its coefficients ELSE reset them to 1
  base.init_addons()
```

**Invariants** — the "no scope attached" branch reads `scope_zoom_factor` into the *iron
sight* factor. That is not a mistake: for a weapon with no detachable optic, the section's
`scope_zoom_factor` **is** its iron-sight field of view. The two names for one number are
frozen by the shipped data.

The silenced sound is selected by whether a silenced sample exists, not by whether a
silencer is attached, so a weapon with a permanent silencer and no dedicated sample keeps
its normal report.

### Silencer coefficients

**Contract** — six multipliers read once from the silencer's section and applied as a
unit when it is attached:

| Coefficient | Clamp | Effect |
|---|---|---|
| hit power | [0, 1] | the round hits softer |
| hit impulse | [0, 1] | and pushes less |
| bullet speed | [0, 1] | and flies slower |
| fire dispersion | **[0, 3]** | the cone, which may widen |
| camera dispersion | [0, 1] | less camera kick |
| camera dispersion increment | [0, 1] | and less growth within a burst |

Five are clamped to at most 1 — a silencer can only take away. The sixth is not, because
some silenced variants are deliberately less accurate.

**Notes** — the coefficients are read only when the slot is *attachable*, but the clamp
runs unconditionally, so a weapon with a permanent silencer gets the default of 1 for
all six. Whether that is intended is not recoverable.

## Fire modes

**Contract** — a section may list fire modes (`1, 2, -1` meaning single, two-round burst,
full auto). The weapon starts in the **last** listed mode. Cycling is circular in both
directions and only while idle; each cycle sets the burst length from the new mode.

`SwitchMode` is a separate, older toggle between single shot and full automatic that
ignores the mode list entirely; it plays the empty-click sound as its feedback and is
refused unless the weapon is idle and not pending. Both survive because different
callers use each.

On becoming a child of a carrier: an actor carrier gets the current fire mode's burst
length; **any other carrier gets unlimited**. Stalkers always fire in automatic and
control their bursts through their own behaviour instead.

## Zoom overrides

**Contract** — entering zoom replays the idle animation (so the aim pose starts from the
right place), notifies scripts, and installs a *zoom inertion* camera effector on the
actor — the slight sway of a scoped view — seeded with the actor's own random seed so that
two clients agree on the sway. Leaving zoom is refused if not zoomed, then replays idle,
notifies scripts and removes the effector.

## `GetBriefInfo` — the heads-up readout

**Contract** — fills the ammunition panel: rounds in the magazine, fire mode (`A` for
automatic, otherwise the burst length), up to three per-type reserve counts and their
total, and the name and icon of the round *currently in the magazine* — falling back to
the selected type when the magazine is empty. Returns false, meaning "nothing changed",
when the inventory has not been modified since the cached frame — but only after the
magazine count and fire mode have already been written, so those two stay live.

Unlimited ammunition, or a weapon with no ammunition types at all, shows dashes.

**Notes** — the panel has room for exactly three ammunition types. A weapon listing more
shows only the first three in the per-type columns, though the total is also only the
first three's sum — a mismatch a rebuild should resolve by summing all types.

## `install_upgrade_impl`

**Contract** — applies a configuration-section upgrade to this weapon, returning whether
anything applied, and doing nothing when run in test mode (which asks "would this
apply?"). Handles: the fire-mode list (which resets the current mode to the last),
the two Abakan parameters, six sound slots, the two silencer particle names, the
silenced shot sound, and the zoom factors — where, exactly as in `InitAddons`, the
section's `scope_zoom_factor` is routed to the scope factor when a scope is attached and
to the iron-sight factor otherwise.

## Sounds

**Contract** — the sound set is loaded by name from the section: draw, holster, shot
(layered), empty click, reload, and two optional variants — reload-from-empty and
reload-after-jam. The two optional ones fall back to the plain reload when the section
does not carry them, which is how the older games' data still works.

`PlayReloadSound` picks between the three: the jam variant if jammed, the empty variant
if the magazine was empty, the plain one otherwise — each falling back to plain when
absent.

`UpdateSounds` re-anchors the positional sounds at the muzzle, at most once per frame.
The shot and empty-click sounds are deliberately *not* re-anchored: they are fired and
forgotten at the position they started at, because a sound that tracks a moving muzzle
sounds wrong.

## Animations

**Contract** — each state has one animation entry point, and each probes **two** names —
a newer and an older one — because the three shipped games name their animations
differently. The reload animation additionally probes for the jam and empty variants and
falls back to the plain reload. The idle animation branches to the aim idle while zoomed.

**Notes** — the two-name probing is the single most pervasive piece of data compatibility
in the weapon code, and a rebuild must keep both name sets for every animation slot.

## `save` / `load`

**Contract** — after the base weapon's payload: burst length, rounds fired, and current
fire mode index, in that order. Loading re-applies the burst length through the setter.
Note `shots_fired` is saved: a save taken mid-burst restores mid-burst.

## `net_Export` / `net_Import`

**Contract** — one extra byte after the base weapon's payload: the current fire mode
index. On import the burst length is re-derived from it, so a remote weapon's cadence
matches its owner's selector.

## `GetWeaponDeterioration`

**Contract** — a single shot wears the weapon by the single-shot figure; anything in a
burst wears it by the burst figure. The test is literally `shots_fired == 1`, so the
first round of a burst counts as a single shot and every later one as a burst round.
