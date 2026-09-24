# src/xrGame/ShootingObject.cpp

> Everything a thing that fires shares: the rate of fire, the damage curve over difficulty, the dispersion cone, the muzzle light, the muzzle flash, smoke, tracer and casing effects, the silencer's multipliers, and the rule deciding which machine is allowed to compute a shot's damage.

**Needs** — [`ShootingObject.h`](ShootingObject.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Actor.h`](Actor.h.md) · [`Spectator.h`](Spectator.h.md) · [`Level.h`](Level.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_single.h`](game_cl_single.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrEngine/Render.h`](../xrEngine/Render.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: parameter parsing, effect lifetimes and a dynamic light; nothing byte-level

## Purpose

Three kinds of thing in the game fire projectiles: a weapon in a creature's hands, a
weapon bolted to a vehicle or helicopter, and a fragmentation grenade scattering
shrapnel. They share no base class otherwise, so this is a mixin holding the parts of
"firing" that are identical for all three:

- the numbers a shot is made of, read from a configuration section;
- the difficulty scaling of damage, which is subtle and entirely data-driven;
- the cosmetic consequences of a shot — light, flame, smoke, tracer, ejected casing —
  each with its own lifetime rule;
- the authority rule: on which machine a given shot's damage is computed, so that in
  multiplayer it is computed exactly once.

It does not own the projectile. Firing hands a bullet to the level's ballistics manager
and forgets it.

## State

```text
RECORD ShootingObject
  # rate of fire
  shot_interval        : real          # seconds between shots, = 60 / rounds-per-minute
  mode_2_shot_interval : real          # the same for a second, faster fire mode
  cycles_down          : bool          # whether the fast mode drops to the normal one after two shots
  shot_timer           : real          # counts down to the next permitted shot
  firing               : bool

  # damage
  hit_power            : real[4]       # one per difficulty level; see LoadFireParams
  hit_power_critical   : real[4]
  hit_impulse          : real
  bullet_speed         : real          # muzzle velocity
  fire_distance        : real          # beyond which the bullet is dropped
  dispersion_base      : real (radians)
  air_resistance       : real          # multiplier on the ballistics model's drag

  # the accurate-first-shot rule
  uses_aim_bullet      : bool
  time_to_aim          : real
  last_shot_time       : real          # global clock of the previous shot; 0 = never fired

  # the shot just fired, kept so the effects can be placed without re-deriving them
  current_shot_pos     : vector
  current_shot_dir     : vector
  current_parent_id    : entity identifier

  # silencer multipliers: applied to hit power, impulse, bullet speed,
  # fire dispersion, camera dispersion and camera dispersion growth
  silencer_when_fitted : Multipliers   # loaded from the silencer
  silencer_current     : Multipliers   # all ones when nothing is fitted

  # muzzle light
  light                : optional<DynamicLight>
  light_base_color, light_base_range      : colour, real
  light_color_variance, light_range_variance : real
  light_lifetime, light_remaining         : real
  light_frame          : int             # the frame the light was last (re)started
  light_enabled        : bool

  # effect names, read once from the section
  flame_particles, smoke_particles, shot_particles, shell_particles : text
  flame_particles_current, smoke_particles_current : text   # swappable per fire mode
  shell_ejection_point : vector          # in the weapon's own space
  flame_effect         : optional<Effect> # the only effect instance held; the rest are fire-and-forget
```

**Invariants**

- The four-element damage array is indexed by difficulty and its element zero is the
  hardest setting. Every element is filled at load even when the data supplies one value.
- The silencer multipliers are all exactly one when no silencer is fitted, so the shot
  path can multiply unconditionally and never branch.
- The muzzle light is created on first use and is *shared across shots* — a burst does not
  stack lights.

## `Load`

**Contract** — reads every shot parameter from a configuration section. Fails on a missing
required key; several keys are optional with stated defaults.

```text
FUNCTION Load(section)
  light_enabled <- IF section has "light_disabled" THEN NOT that ELSE true

  rate       <- section."rpm"                       # rounds per minute; required
  rate_mode2 <- section."rpm_mode_2" OR rate        # a second, usually faster, fire mode
  REQUIRE rate > 0
  shot_interval        <- 60 / rate                 # seconds per shot
  mode_2_shot_interval <- 60 / rate_mode2
  cycles_down <- section."cycle_down" OR false      # see note

  LoadFireParams(section)
  LoadLights(section, prefix: "")
  LoadShellParticles(section, prefix: "")
  LoadFlameParticles(section, prefix: "")

  air_resistance <- section."air_resistance_factor" OR 1
```

**Notes** — the second fire rate and the cycle-down flag exist for one weapon in the
series: a rifle whose first two rounds of a burst leave at a far higher cyclic rate than
the rest, so that both are in the air before recoil moves the muzzle. Modelling it needs
two rates and a rule for switching between them after two shots, and that is exactly what
these two keys are. A rebuild that wants the weapon to feel right must keep it.

Every loader takes a *prefix*, which is prepended to each key name. That is how one
section describes two independent firing systems — a rifle and its underbarrel launcher —
with `flame_particles` and `grenade_flame_particles` in the same section. The prefix is
empty for the primary system.

## `LoadFireParams`

**Contract** — reads damage, impulse, range, muzzle velocity and dispersion, and expands
the damage values across the four difficulty levels.

```text
FUNCTION LoadFireParams(section)
  dispersion_base <- radians(section."fire_dispersion_base")

  # damage is a comma-separated list, hardest first
  power_list    <- section."hit_power"                          # required
  critical_list <- section."hit_power_critical" OR power_list   # defaults to the same curve

  FOR EACH list, target IN ((power_list, hit_power), (critical_list, hit_power_critical))
    target[hardest] <- number(list[0])
    target[veteran] <- target[stalker] <- target[novice] <- target[hardest]  # flat by default
    IF list has > 1 entries THEN target[veteran] <- number(list[1])
    IF list has > 2 entries THEN target[stalker] <- number(list[2])
    IF list has > 3 entries THEN target[novice]  <- number(list[3])

  hit_impulse    <- section."hit_impulse"
  fire_distance  <- section."fire_distance"
  bullet_speed   <- section."bullet_speed"
  uses_aim_bullet <- section."use_aim_bullet"
  IF uses_aim_bullet THEN time_to_aim <- section."time_to_aim"
```

**Invariants** — the list order is **hardest to easiest**, and a list shorter than four
means "the same on every remaining difficulty". This is frozen by the shipped
configuration: several hundred weapon sections use one, two or four entries and all three
shapes must read identically. A rebuild that orders the list the other way silently makes
the game hardest on the easiest setting.

**Notes** — the critical-damage curve defaulting to the ordinary one rather than to zero
matters: most shipped sections omit it, and a zero default would remove critical hits from
almost every weapon in the game.

## `FireBullet`

**Contract** — fires one round. Perturbs the aim direction inside a cone of the given
dispersion, records the shot so the effect routines can place themselves, decides whether
this round is an "aimed" round, selects a damage value for the shooter and difficulty,
applies the silencer multipliers, and hands the whole thing to the level's ballistics
manager. Does not block, does not allocate, and does not itself resolve any hit — the
ballistics manager traces the bullet over subsequent frames.

```text
FUNCTION FireBullet(pos, aim_dir, dispersion, cartridge,
                    shooter_id, weapon_id, i_compute_the_hit, shot_index)
  dir <- random direction within a cone of half-angle `dispersion` about aim_dir
  current_shot_pos <- pos ; current_shot_dir <- dir ; current_parent_id <- shooter_id

  # the accurate round
  aimed <- false
  IF uses_aim_bullet AND shooter may have aimed rounds THEN
    aimed <- (last_shot_time = never) OR (now - last_shot_time >= time_to_aim)
  last_shot_time <- now

  # difficulty applies only to the player, and only in single player
  IF shooter is the player AND game is single player THEN power <- hit_power[current_difficulty]
  ELSE                                                     power <- hit_power[hardest]

  ballistics.add_bullet(
      origin: pos, direction: dir,
      speed:   bullet_speed * silencer_current.bullet_speed,
      power:   power        * silencer_current.hit_power,
      impulse: hit_impulse  * silencer_current.hit_impulse,
      shooter_id, weapon_id, kind: firearm_wound,
      max_distance: fire_distance, cartridge, air_resistance,
      authoritative: i_compute_the_hit, aimed, shot_index)
```

**Invariants**

- Difficulty scaling applies **only** to shots the player fires, and only in single
  player. Every creature, and every shot in multiplayer, uses the hardest column. Without
  that asymmetry, lowering the difficulty would make enemies weaker *and* the player's own
  weapons weaker, since both read the same section.
- The previous-shot time is recorded on every shot regardless of whether the round was
  aimed, so that "aimed" means "enough time since I last fired", which is a proxy for
  "I have been holding still and looking down the sights". A rebuild that instead tracks
  actual aiming state changes the feel; this is a deliberate approximation.
- The first round ever fired from a weapon is always aimed, because there is no previous
  shot. That is how a single carefully placed opening shot is rewarded.

## `SendHitAllowed`

**Contract** — answers "should *this* machine compute the damage for a shot fired by this
user?". Exactly one machine in a session must answer yes for any given shot, or damage is
applied twice or not at all.

```text
FUNCTION SendHitAllowed(user) -> bool
  IF the game mode computes hits on the server THEN RETURN running_as_server

  IF running_as_server THEN
    IF user IS a player THEN RETURN user IS the locally controlled entity
    RETURN true                      # creatures are always the server's
  ELSE
    IF user IS a player THEN RETURN user IS the locally controlled entity
    RETURN false
```

**Invariants** — outside server-authoritative mode, the rule reduces to: *the machine
whose human pulled the trigger computes the hit*, and creatures belong to the server.
Lag then costs the shooter nothing — he sees his own hits immediately — at the price of
trusting the client, which is why the alternative mode exists and why it is a per-mode
setting rather than a constant.

## `Light_Start` / `Light_Render` / `UpdateLight` / `StopLight` / `RenderLight`

**Contract** — the muzzle flash light. Started on a shot, rendered at the muzzle each
frame while it lives, faded out over its configured lifetime, and deactivated when it
expires. The light object itself persists between shots and is only created on first use.

```text
FUNCTION Light_Start()
  IF light IS none THEN create it, shadow-casting only on a deferred renderer
  IF light_frame = current_frame THEN RETURN      # at most one restart per frame
  light_frame     <- current_frame
  light_remaining <- light_lifetime
  light_build_color <- each channel of light_base_color jittered by light_color_variance
  light_build_range <- light_base_range jittered by light_range_variance

FUNCTION UpdateLight()
  IF light EXISTS AND light_remaining > 0 THEN
    light_remaining <- light_remaining - frame_delta
    IF light_remaining <= 0 THEN light.active <- false

FUNCTION Light_Render(muzzle_position)
  LOCK render_lock DURING
    fade <- light_remaining / light_lifetime
    light.position <- muzzle_position
    light.color    <- light_build_color * fade
    light.range    <- light_build_range * fade
    light.active   <- true
```

**Invariants**

- The once-per-frame guard is the reason a weapon firing several rounds in one frame — a
  shotgun, a very high rate of fire, a frame hitch — produces one flash and not a stack of
  them with a stale fade.
- Colour *and* range fade together and linearly. Fading only the colour leaves a
  full-radius dark light that still costs the renderer the same; fading only the range
  makes the flash pop out.
- Shadow casting is enabled only on the deferred renderers. On the oldest path a
  shadow-casting light per muzzle flash is unaffordable, and the flash is short enough
  that its absence is not noticed.

**Notes** — the render step is serialized against itself because the light is written from
the render path while the simulation may be starting a new flash. The problem it solves is
a torn light — position from one frame and colour from another — not a crash. A rebuild
that produces the light's parameters as immutable per-frame data and hands them to the
renderer has no lock and no problem.

## `StartParticles` / `UpdateParticles` / `StopParticles`

**Contract** — the shared lifecycle for any of this object's effects. `StartParticles`
fills a caller-owned slot: an occupied slot is merely repositioned; an empty one gets a
newly created effect placed at a position with an inherited parent velocity and played.
`UpdateParticles` repositions an existing effect and destroys it if it is a one-shot that
has finished. `StopParticles` stops and destroys immediately.

The effect's orientation comes from the owning object's particle transform and only its
*origin* is overridden with the supplied position — muzzle smoke must point where the
barrel points.

An effect may be marked auto-removing, in which case the particle system frees it when it
finishes and the caller must not hold the slot.

**Invariants** — the decision every start makes is whether the effect plays in
**first-person space** or in world space, and it is not simply "is the player holding
this":

```text
FUNCTION plays_in_first_person_space() -> bool
  hud <- this object is currently drawn as a held, first-person model
  IF hud AND the controlled entity is a spectator NOT in first-eye mode THEN hud <- false
  IF running in the oldest game's compatibility mode THEN hud <- false
  RETURN hud
```

The spectator clause exists because a spectator watching another player through a
free-flying or chase camera still has that player's weapon "in hand" as far as the weapon
knows; playing its muzzle flash in first-person space would paste it over the camera.

The compatibility clause is data archaeology: the first game's effect offsets were
authored against world-space placement, and honouring first-person space puts every
muzzle flash in the wrong place for that game's content. It is a per-game data quirk, not
an engine one, and a rebuild that reauthors the effects can drop it.

## `StartFlameParticles` / `UpdateFlameParticles` / `StopFlameParticles`

**Contract** — the muzzle flame is the one effect this object *keeps*, because a
continuously firing weapon must show one sustained flame rather than a new one per shot.

Starting a flame while a looping flame is already playing only repositions it. Otherwise
any existing flame is released and a new one is created at the current muzzle point.
Stopping does not destroy the effect: it marks it auto-removing, stops emission, and drops
the reference, so the particles already in the air finish their own lives and the effect
frees itself. Destroying it outright would make the flame vanish mid-air the instant the
trigger is released.

Updating repositions the flame at the muzzle each frame and reaps a finished one-shot
flame.

## `StartSmokeParticles` / `StartShotParticles` / `OnShellDrop`

**Contract** — the three fire-and-forget effects. Each creates an auto-removing effect and
never holds a reference:

- smoke, at a caller-supplied position with the shooter's velocity inherited;
- the tracer/shot trail, placed at the shot's recorded origin and pointed along its
  recorded direction;
- the ejected casing, at the ejection point with the shooter's velocity inherited.

**Invariants** — the casing effect is **skipped entirely beyond two metres from the
camera**. A casing is a few centimetres of tumbling metal that nobody sees from further
away, and creatures firing across a level would otherwise spawn an effect per round for
every shooter on the map. This is the only distance culling in the file and it is the one
that pays.

## `LoadLights` / `LoadShellParticles` / `LoadFlameParticles`

**Contract** — read the light and effect parameters from a section under a key prefix.
Light parameters are read only when the light is enabled. Each effect name is optional and
an absent key simply leaves that effect off. The casing ejection point is read only when a
casing effect is named, since it is meaningless otherwise. The two swappable effect names
— flame and smoke — are initialized to their loaded values, and a fire-mode change
rewrites the *current* pair without disturbing the loaded one.

## `FireStart` / `FireEnd` / `IsWorking`

**Contract** — set and clear the "currently firing" flag and report it. The flag drives
the sustained effects and the animation layer; the actual round timing is the shot timer's.

## `Light_Create` / `Light_Destroy`

**Contract** — allocate and release the renderer's light object. Creation chooses shadow
casting by renderer generation, as above.

## `reinit`

**Contract** — clears the held flame effect reference without stopping it. Called when the
object is being re-spawned into a new life; the effect itself belongs to the previous
level's particle system, which has already been torn down.

## `SetBulletSpeed` / `GetBulletSpeed`

**Contract** — read and write the muzzle velocity, so that an upgrade or a script can
change it after load.

## `ParentMayHaveAimBullet` / `ParentIsActor`

**Contract** — two questions about the shooter that this mixin cannot answer itself,
defaulting to no. A weapon in a creature's hands overrides them from its owner; a vehicle
gun leaves them false, so vehicle weapons never get aimed rounds and never scale by
difficulty.

## `IsHudModeNow` / `get_CurrentFirePoint` / `get_ParticlesXFORM` / `ForceUpdateFireParticles`

**Contract** — what the mixin demands of whatever it is mixed into: whether the object is
currently drawn as a first-person model, where its muzzle is in world space, the transform
effects are oriented by, and an optional hook to force the effects to catch up with a pose
change within a frame. The first three have no default and must be supplied.

## Notes

Two named constants are defined in the file and used nowhere in it — a minimum hit power
below which a hit is discarded, and the size of a bullet-hole decal. They belong to the
ballistics manager and travelled here by inheritance of an old file split. A rebuild
should define them where they are read.

`DumpActiveParams` is declared here and implemented separately, in the anti-cheat dump
module: it writes this object's live parameter values back out in configuration form, so
that a server can compare a client's effective weapon numbers against the shipped section
and detect tampering. Keeping it out of this file keeps the dump format's churn away from
the firing path.
