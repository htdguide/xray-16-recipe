# src/xrGame/ai/monsters/pseudogigant/pseudo_gigant.cpp

> A creature that fights with the ground: every footfall shakes the camera by an amount that falls off with distance, and its stomp is a timed, ranged area attack that throws loose physics objects, staggers the player's movement and lands a hit that weakens with distance.

**Needs** — [`pseudo_gigant.h`](pseudo_gigant.h.md) · [`pseudo_gigant_step_effector.h`](pseudo_gigant_step_effector.h.md) · [`pseudogigant_state_manager.h`](pseudogigant_state_manager.h.md) · [`../ai_monster_effector.h`](../ai_monster_effector.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../../../Actor.h`](../../../Actor.h.md) · [`../../../Level.h`](../../../Level.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Networking transport](../../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`pseudo_gigant.h`](pseudo_gigant.h.md)
**Tier floor** — T2: applies impulses to rigid bodies and emits a damage event through the authoritative record stream

## Purpose

Three distinctive mechanics, all of them about weight.

**The footfall shake.** The animation layer fires a callback at the frame each foot lands; the
giant answers it by pushing a camera effect whose strength falls off with the player's distance.
This is a *presence* mechanic rather than a combat one — a giant walking somewhere nearby is
felt before it is seen.

**The stomp as a gated ability, not a state.** The stomp is not a node in the brain's state
tree. It is a motion-control request that the shared base offers and this creature vetoes or
permits, which is why the giant's brain is the plain baseline selector. Range band, cooldown
and the existence of an enemy are all checked in the veto.

**Distance-scaled everything.** The stomp's damage, camera shake and screen effect are all
scaled by the same falloff factor, computed once. A distant player is nudged; a player standing
on the giant's foot is thrown.

## `Load`

**Contract** — reads the entire creature from a configuration section: the animation table with
its velocity profiles and footfall effect names, the three step-shake numbers, the stomp's
screen-effect description from a *named second section*, two sounds, the stomp's damage,
particle asset, range band, cooldown band, and the stagger duration. Every key is mandatory.
Allocates sound and animation handles. Ends by handing the section to the shared base's
post-load pass.

**Notes** — two things in the animation table are distinctive and both are deliberate. The
damaged-variant substitution registers a *predicate plus a pair*: while the "damaged" flag is
set, requests for the run and walk animations silently resolve to their damaged versions, so
the brain never has to know the creature is hurt. And the giant's run animation is registered
as the *same clip and the same velocity profile as its walk* — the giant has no run. The
motion-speed table that would have given running its own speed is commented out beside it.

The stomp's screen effect is described in a second configuration section named by the first,
following the chapter's standard split: the creature's section says which effect, a separate
block says what it looks like. Its colour fields are comma-separated text parsed at load,
because the configuration format has no vector type.

## `reinit`

**Contract** — reloads the two leap velocity profiles from the section, registers the
rotation-jump animation quadruple with a quarter-turn step, and binds the stomp to its
animation and the fraction of that animation at which the hit lands. Runs on every spawn and
every save load.

**Notes** — the hit fraction (a little under half the clip) is the giant's contract with its
own animation: the stomp's effects fire when the foot visually touches the ground, not when
the state was entered. A rebuild retiming the animation must retime this number with it.

The leap-attack data is present but commented out, and the corresponding ability is not
registered in the constructor — the giant has a rotation jump but no leap attack, even though
`HitEntityInJump` below is written for one.

## `event_on_step`

**Contract** — called from the animation layer at each footfall. If the player is within the
hard-coded sixty-unit radius, pushes a camera-shake effect whose power is the normalised
remaining distance. No effect at all beyond the radius. Called often; must stay cheap.

```text
FUNCTION event_on_step()
  player = the entity the camera is attached to
  IF player IS none
    RETURN
  d = distance(player, self)
  IF d < 60
    power = (60 - d) / (1.2 * 60)     # 0 at the rim, about 0.83 at the giant's own feet
    push_camera_shake(step_shake.time, step_shake.amplitude, step_shake.periods, power)
```

**Notes** — the sixty-unit radius is compiled in, not authored, which is the one giant number
a modder cannot touch. The divisor of 1.2 caps the power below one even at zero distance — the
shake is deliberately never at full amplitude, presumably because full amplitude was
unplayable. Neither number has a recoverable derivation.

## `check_start_conditions` — the stomp gate

**Contract** — answers whether a motion-control request may seize the creature. Consults the
shared base first, then applies the giant's own rules. This is the only place the stomp's
preconditions live.

```text
FUNCTION check_start_conditions(kind) -> bool
  IF NOT base.check_start_conditions(kind)
    RETURN false
  IF kind == run_attack
    RETURN true                        # always allowed; bypasses everything below
  IF kind == stomp
    IF now < next_stomp_allowed_at     RETURN false
    IF no enemy                        RETURN false
    d = distance(enemy, self)
    IF d > stomp_range.max OR d < stomp_range.min   RETURN false
  RETURN true
```

**Invariants** — the **minimum** range is what makes the giant readable: too close and it
cannot stomp, so it must disengage and re-approach, which is the rhythm of the fight. The
cooldown is checked here rather than in the brain, so a giant whose stomp is on cooldown simply
never gets offered it and keeps doing whatever its selector chose.

## `on_activate_control`

**Contract** — a motion-control request was granted. For the stomp: plays the wind-up sound at
the creature's head and immediately samples the next cooldown from the authored band. The
cooldown is armed at *activation*, not at the hit, so a stomp that is interrupted still costs
the giant its cooldown.

## `on_threaten_execute` — the stomp lands

**Contract** — fired by the animation at the stomp's hit frame. Throws nearby physics objects,
plays the impact sound and particles, then — only if the enemy is the player and the player is
not airborne — applies a distance-scaled camera shake, screen effect, camera wrench, movement
lock and damage event. Blocks on nothing; emits one damage event into the authoritative record
stream.

```text
FUNCTION on_threaten_execute()
  FOR EACH object WITHIN 15 units
    IF object HAS a physics body
      direction = normalize((object.position + 2 units up) - self.position)
      object.apply_impulse(direction, 20 * object.mass)   # upward-biased, mass-proportional

  play_sound(impact, at: self.position slightly raised)
  play_particles(stomp_particles, at: same point, facing: self.direction)

  IF enemy IS NOT the player       RETURN
  IF the player is mid-jump        RETURN                # jumping over the shockwave works

  falloff = clamp(stomp_damage - stomp_damage * distance(player, self) / stomp_range.max, 0, 1)

  push_camera_shake(stomp_effector.shake scaled by falloff)
  push_screen_effect(stomp_effector.screen scaled by falloff)
  player.camera.nudge(random horizontal direction, random amount up to 0.3 * falloff)
  player.camera.nudge(random vertical direction,   random amount up to 0.3 * falloff)
  player.lock_acceleration_for(stagger_duration)
  emit_damage_event(target: player, from: self, direction: straight up,
                    power: falloff, bone: the player's root,
                    impulse: 80 * player.mass, kind: strike)
```

**Invariants**

- **The impulse is proportional to each object's own mass**, so a barrel and a crate fly the
  same way rather than the heavy one barely moving. The direction is computed to a point two
  units *above* each object's centre, which biases everything upward — objects are thrown, not
  shoved sideways.
- **Jumping is the counter.** The airborne check is the one skill expression in the fight and a
  rebuild must keep it; without it the stomp is unavoidable.
- **The falloff factor drives all four effects**, so damage, shake, screen distortion and camera
  wrench always agree. The expression is a linear falloff over the stomp's *maximum* range,
  then clamped into zero-to-one — which means the authored damage value doubles as the
  upper bound on the effect strength, and authoring it above one flattens the falloff near the
  centre.
- **The damage event's direction is straight up and the impulse is proportional to the
  player's mass**, so the player is launched rather than pushed away — the hit reads as the
  ground heaving.
- The object sweep radius (fifteen units), the impulse coefficients (twenty for objects, eighty
  for the player), the two-unit upward bias and the 0.3 camera nudge are all compiled in.

## `HitEntityInJump`

**Contract** — applies damage when a leap connects, reading power, impulse and impulse
direction from the *leap animation's* own authored parameter block rather than from the
creature's section. Damage that belongs to a move is authored with the move.

**Notes** — the giant never leaps: the leap ability is commented out of the constructor and its
data out of `reinit`. This entry point is reachable only if a rebuild re-enables them.

## `TranslateActionToPathParams`

**Contract** — converts the currently requested action into the velocity masks the path builder
plans with. For anything but walking or running, defers to the shared base. For walking and
running, it selects the *walk* profiles in both cases, picking the damaged variants when the
creature is hurt, and enables the path.

**Notes** — this is the giant's defining movement decision and it is enforced here rather than
in the animation table alone: **asking a giant to run gets you a walk.** The "desirable" mask
names a single profile while the "allowed" mask names a family, which lets the builder slow
below the desired speed when geometry demands it but never speed up past a walk. A flag on the
creature ("force real speed") collapses the two masks together, pinning the giant to exactly one
speed — used when something else is driving the movement and the builder must not improvise.
