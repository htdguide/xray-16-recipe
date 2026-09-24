# src/xrGame/ai/monsters/anti_aim_ability.cpp

> A creature that notices it is being aimed at charges up, then lurches the player's camera aside — the mechanic that makes a burer hard to shoot.

**Needs** — [`anti_aim_ability.h`](anti_aim_ability.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`Actor.h`](../../Actor.h.md) · [`ActorEffector.h`](../../ActorEffector.h.md) · [`Inventory.h`](../../Inventory.h.md) · [`Weapon.h`](../../Weapon.h.md)
**Used by** — [`anti_aim_ability.h`](anti_aim_ability.h.md)
**Tier floor** — T2: an angle integrator driving a camera effector with real-time deadlines

## Purpose

This is the clearest example in the chapter of a creature ability that reaches *out of the
simulation and into the player's controls*. The creature measures how precisely the player's
crosshair is on it, integrates that into a detection level, and when the level fills it
plays an animation and launches a camera effector that shoves the aim away.

Everything about it is asymmetric on purpose: the level rises fast and falls slowly, so
brief aiming is punished and a player must look away for a while to reset it. The whole
ability only applies against **the player** — it checks that its enemy is the player and
that the player is holding a weapon — because there is nothing to disturb otherwise.

## State

```text
RECORD AntiAimAbility
  creature            : reference

  # settings, all from the creature's configuration section
  timeout             : real = 5      # seconds between uses
  freeze_time         : real = 1      # loaded and never read (see Notes)
  max_angle           : real = 0.5    # radians; beyond this the player is not aiming at me
  gain_speed          : real = 1      # detection units per second at perfect aim
  decay_speed         : real = 0.1    # detection units per second, always subtracted
  effectors           : list<text>    # named camera effectors to choose among

  # state
  detection_level     : real [0..1]
  last_angle          : real          # previous update's aim error, for smoothing
  last_detection_tick : int
  last_activated_tick : int
  animation_hit_tick  : int           # when within the animation the effect lands
  animation_end_tick  : int
  effector_end_tick   : int
  effector_id         : int           # 0 = no effector running
```

**Invariants** — the detection level is clamped to zero-to-one and reaching one is the
trigger. It is reset to zero on deactivation, so every use starts from cold. An effector
identifier of zero means none is running, which is why the identifier is allocated from the
camera manager rather than being a flag.

## `update_schedule`

**Contract** — the driver. Runs one of three paths: abort if the preconditions no longer
hold, run the activation sequence if active, sit out the cooldown, or integrate the
detection level. Reaches the player and the player's camera directly.

```text
FUNCTION update_schedule()
  IF NOT preconditions_hold() THEN
    deactivate() ; RETURN

  IF start_condition() THEN activate_through_control_manager()

  IF active THEN
    IF now < animation_hit_tick THEN RETURN      # the wind-up is still playing
    IF no effector running THEN launch_effector()
    IF now < effector_end_tick THEN RETURN       # the effector is still running
    deactivate()

  IF now < last_activated_tick + timeout THEN RETURN    # cooldown; do not charge

  delta = (now - last_detection_tick) / 1000 ; last_detection_tick = now

  angle         = aim_error()
  smoothed      = min(max_angle, (angle + last_angle) / 2)
  closeness     = (max_angle - smoothed) / max_angle          # 1 at dead centre, 0 at the edge
  gain          = can_see_enemy_roughly() ? closeness^2 * gain_speed : 0

  detection_level = clamp(detection_level + (gain - decay_speed) * delta, 0, 1)
  last_angle      = angle
```

**Invariants** — the decay is subtracted **unconditionally**, including while the player is
aiming perfectly. So the effective charge rate is `closeness² × gain − decay`, and with the
shipped defaults a player whose aim error exceeds about a third of the maximum angle never
charges the ability at all. That is the real threshold of the mechanic, and it is not
stated anywhere — it falls out of the two speeds.

The gain is **quadratic** in closeness, which sharpens the same threshold: the ability
responds to precise aim, not to being vaguely pointed at.

**Notes** — the smoothing is a two-sample mean of the current and previous aim error, which
costs one update of latency and removes the flicker a player's natural hand movement would
otherwise produce. Taking the minimum with the maximum angle before computing closeness is
what keeps the closeness term non-negative without a clamp.

The cooldown check sits *after* the active block, so an ability that has just finished
begins its cooldown from the moment it activated, not from the moment it ended. The effect
duration is therefore inside the timeout rather than added to it.

## `aim_error`

**Contract** — how far the player's view direction is from the creature, measured in
radians *beyond the creature's own angular size*. Returns half a turn — the maximum — when
the creature cannot currently see the player, which drives the level straight down.

```text
FUNCTION aim_error() -> real
  IF NOT creature.can_see(player) THEN RETURN PI        # not aimed at me at all

  to_centre = creature.centre       - player.head_position
  to_head   = creature.head_position - player.head_position

  angular_radius = angle_between(to_centre, to_head)    # the creature's apparent size
  deviation      = angle_between(to_centre, player.camera_direction)

  RETURN max(0, deviation - angular_radius)
```

**Invariants** — subtracting the creature's angular radius is what makes the mechanic
size-aware and distance-aware in one step: anywhere on the creature's silhouette counts as
perfect aim, and a distant creature presents a smaller target that must be aimed at more
precisely. Approximating the silhouette by the centre-to-head angle is crude and is the
only geometry available without a real projection.

## `preconditions_hold`

**Contract** — the ability is meaningful only when the creature is alive, the player exists
and is alive, **the creature's current enemy is the player**, and the player has a weapon
in hand. Any failure tears the ability down.

**Notes** — the weapon test is the one that reads as arbitrary and is not: with no weapon
there is no aim to disturb, and charging the ability against an unarmed player would
produce a camera lurch with no cause the player could understand.

## `start_condition`

**Contract** — a stack of refusals. Refuses when already active; when the creature is under
script control and the ability has not been explicitly forced; when the creature's own
ability arbitration says no; when any of the animation, path or movement channels is held
by something else; when the preconditions fail; when an override animation is playing; when
the detection level is not yet full (unless forced); and when the creature is not roughly
facing its enemy.

**Notes** — the forced path exists so a script can fire the ability regardless of charge.
The "roughly facing" test admits anything within 70 degrees, which is much looser than the
aim test and exists only to stop the creature playing the animation with its back turned.

## `activate`

**Contract** — seizes the animation, path and movement channels and halts the latter two,
so the creature stops dead for the ability. Starts the ability's animation and computes two
deadlines from the *animation data*: when within the motion the effect lands, and when the
motion ends. Clears the forced flag.

**Invariants** — the hit moment comes from the animation's own hit timing (the same data
melee attacks use), so the camera lurch is synchronised to the visible gesture. That is why
the effector is launched from the update loop at a deadline rather than at activation.

## `launch_effector`

**Contract** — picks one of the configured effectors at random, allocates a camera-effector
identifier from the player's camera manager, constructs a non-cyclic animator-driven
effector from the named configuration, starts it, and adds it to the player's camera. Sets
the effector's end deadline to the later of the effector's own length and the animation's
end. Fires the owner's hit callback.

**Invariants** — the end deadline being the **maximum** of the two is what guarantees the
ability does not release its channels while either the animation or the camera disturbance
is still running. Either can be the longer one depending on the creature's data.

**Notes** — the effector is configured entirely by name from data: the configuration section
named in the ability's effector list supplies the animation file and whether the effect also
moves the first-person weapon view. So a creature's anti-aim *feel* is authored, not coded.

The random choice is over the whole list each time, so consecutive uses can repeat.

## `deactivate`

**Contract** — releases the three channels, unsubscribes from animation events, removes the
camera effector from the player, clears the effector identifier, and resets the detection
level and the smoothing sample to their cold values.

**Notes** — the guard at the top of this routine tests `is_active` and returns *when it is
true*, which is inverted relative to every other guard in the file. It works only because
the single caller deactivates through the control framework first, so by the time this runs
the component is already inactive. A rebuild should not reproduce the inversion.

The teardown path checks that the creature still has a visual before asking the control
framework to deactivate, because a creature being destroyed may have released its model
already. That is the incidental problem — a lifetime ordering — and the decision behind it
is that **the camera effector must be removed from the player even when the creature is
gone**, which is why the destructor calls this at all.

## `on_creature_death`

**Contract** — tears the ability down. Without it, a creature killed mid-ability leaves the
player's camera shaking with no source.

## Configuration keys

`anti_aim_timeout`, `anti_aim_effectors` (a comma-separated list of section names),
`anti_aim_freeze_time`, `anti_aim_max_angle`, `anti_aim_detection_gain_speed`,
`anti_aim_detection_loose_speed` — all optional, all with the defaults listed under State.

**Notes** — `anti_aim_freeze_time` is loaded and never read. It was presumably meant to
hold the creature still for a period after the effect; the channel capture does that
instead, for the animation's duration. Its value in the shipped data has no effect.
