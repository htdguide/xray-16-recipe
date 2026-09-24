# src/xrGame/ai/monsters/burer/burer.cpp

> The burer's definition and its three physical abilities: a gravity wave that walks across the floor toward its victim, a bullet shield that eats fire-wounds, and a stamina drain that knocks the weapon out of the player's hands.

**Needs** — [`burer.h`](burer.h.md) · [`burer_state_manager.h`](burer_state_manager.h.md) · [`burer_fast_gravi.h`](burer_fast_gravi.h.md) · [`base_monster.h`](../basemonster/base_monster.h.md) · [`telekinesis.h`](../telekinesis.h.md) · [`anti_aim_ability.h`](../anti_aim_ability.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`control_direction_base.h`](../control_direction_base.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`burer.h`](burer.h.md)
**Tier floor** — T1: the wave steps through the world with a ray cast per step and applies impulses to every rigid body it passes, inside the frame update

## Purpose

The burer is the chapter's most configuration-heavy creature: roughly thirty authored numbers across three abilities, and the file is best read as "load those numbers, then implement the three things they tune". Its animation table follows the pattern stated once in [`boar.cpp`](../boar/boar.cpp.md), with one twist described below.

## State

See [`burer.h`](burer.h.md) for the field list. The invariants that matter:

- At most **one** gravity wave exists per creature at a time. `GraviObject.active` is the whole of that guarantee; launching while one is in flight overwrites it.
- The wave holds a reference to its victim, and that reference must be cleared when the victim is destroyed — otherwise the wave chases a dead handle.
- `shield_active` gates both damage and decals: while it is up, fire-wound hits are converted to sparks and no bullet mark is left.
- `can_scan` is a single flag shared by *every* burer in the world, not per creature.

## `Load`

**Contract** — Reads the configuration section: three particle effect names, three sound assets, the gravity block, the telekinesis block, the shield block, the anti-aim block, the run-away geometry — then declares the animation table. Called once per creature. Fails loudly only on the clips and parameters marked required.

The authored numbers, with their code defaults where they have one:

```text
GRAVITY WAVE
  speed, step, radius                   required
  time_to_hold                          required   # wind-up before launch scales with range
  impulse_to_objects, impulse_to_enemy  required
  hit_power                             required
  min_dist                              default 1 world unit
  max_dist                              default radius * 3
  cooldown                              default round(impulse_to_enemy / hit_power * 5)

TELEKINESIS
  max_handled_objects, time_to_hold     required
  object_min_mass, object_max_mass      required
  find_radius                           required
  max_time                              default 10000 ms
  min_distance / max_distance           defaults 8 / 30 world units
  raise_speed / fly_velocity            defaults 5 / 30
  object_height                         default 2 world units

SHIELD
  cooldown / time                       defaults 4000 / 3000 ms
  keep_particle and its period          defaults none / 1000 ms

ANTI-AIM STAMINA DRAIN
  weight_to_stamina_hit                 default 0.02
  weapon_drop_stamina_k                 default 3
  weapon_drop_velocity                  default 8

POSITIONING
  runaway_distance / normal_distance    defaults 6 / 12 world units
  max_runaway_time                      default 5000 ms
```

**Notes** — The gravity cooldown's default is derived rather than chosen: `impulse_to_enemy / hit_power * 5`, rounded. It ties the pause between waves to how *hard* the wave hits relative to how much it hurts — a wave tuned to throw the player a long way without doing much damage gets a long cooldown. A rebuild that hard-codes a default here loses that coupling silently. Likewise the default maximum range is three times the wave's radius, so widening the wave in data also lengthens its reach.

The burer's animation table is the only one in the chapter that is substantially *optional*. Almost every entry is declared as not-required and falls back: if the model has no dedicated shield clip, the first and second sub-clips of the gravity animation stand in; if it has no dedicated telekinesis-fire clip, the third gravity sub-clip does; if there is no damaged variant, no substitution is registered at all. And the gravity attack itself has two shapes — a single-clip form and a three-clip form — decided by which clips the model turns out to have, recorded in a flag the attack state reads. This is what lets one class serve models from three different games.

## `reload`

**Contract** — Registers the two creature-specific sounds — the gravity attack and the telekinetic attack — with the shared sound player, both emitted from the head bone, both classed as attack sounds, at priorities above the shared set. Called on every load of the section, separate from `Load`.

## `UpdateGraviObject` — the travelling wave

**Contract** — Advances the single wave in flight by at most one step per frame. Applies an impulse to every rigid body within its radius, spawns the wave particle at its new position, keeps the wave's looping sound anchored to it, and — if it has arrived at its victim with line of sight — deals the damage and ends. Runs inside the frame update. Casts one ray per advance and does one proximity query per advance.

```text
FUNCTION UpdateGraviObject()
  IF NOT active                                   RETURN
  IF victim is gone or destroyed                  deactivate ; RETURN
  IF travelled beyond the target                  deactivate ; RETURN

  elapsed  = now() - time_last_update
  advance  = elapsed * speed / 1000
  IF advance < step                               RETURN      # accumulate, do not creep

  direction = unit(target_pos - current_pos)
  next_pos  = current_pos + direction * advance

  # Has the wave reached the victim?
  aim = unit(victim_centre - next_pos)
  IF ray_cast(next_pos, aim, length = step) hits the victim within that length
     AND the victim is in this creature's current field of vision
        deal hit_power damage with impulse_to_enemy straight upward,
             typed as a strike
        deactivate ; RETURN

  current_pos      = next_pos
  time_last_update = now()

  spawn the wave particle at current_pos, oriented along `direction`
  FOR EACH rigid body within `radius` of current_pos
     apply an impulse of impulse_to_objects * its mass,
           directed away from current_pos
  anchor the looping wave sound half a unit above current_pos
```

**Invariants** — The wave never moves less than one `step`; the accumulation is what makes the wave's speed independent of frame rate while keeping each step's ray cast meaningful. The wave stops the first time it passes its launch-to-target distance, so it cannot chase a fleeing player indefinitely.

**Notes** — The impulse on the victim is applied *straight up*, not along the wave's travel: the burer's wave lifts you off your feet rather than pushing you back. The impulse on scenery is radial and scaled by each body's mass, so heavy objects get the same acceleration as light ones and the debris moves as one shell. Both are authored separately, which is how a wave can be made to devastate furniture without being lethal, or the reverse.

The victim test is deliberately double: the ray must reach the victim *and* the creature must currently be able to see the victim through its own vision system. A ray alone would let the wave hit through a gap the creature cannot perceive.

## `Hit` — the shield

**Contract** — Intercepts incoming damage. While the shield is up, a fire-wound hit is consumed: no damage is taken and a spark effect is spawned at the impact point, oriented from the hit's bone and direction. Any other damage type while the shield is up is also consumed, silently. With the shield down, damage passes to the base creature. At most one spark effect per frame, whatever the rate of fire.

```text
FUNCTION Hit(hit)
  IF shield_active AND hit.type == fire_wound AND this frame has not sparked yet
     spawn the shield spark at the hit's bone-space position and direction
  ELSE IF NOT shield_active
     base.Hit(hit)
  last_hit_frame = current_frame
```

**Notes** — The frame gate on the spark is a rate limit, not a correctness rule: a burst of automatic fire would otherwise spawn an effect per bullet. The structure of the branch means a *non*-fire-wound hit landing on a raised shield falls through both arms and is dropped entirely — explosions and strikes are absorbed too, but silently. Whether the silence was intended is not recoverable; what is certain is that the shield stops everything, not only bullets.

## `CanDeactivateShieldEarly`

**Contract** — Whether the shield may drop before its authored duration is up. Yes when there is no enemy at all. Yes when the enemy is carrying a weapon and is *reloading* it. No otherwise — including, pointedly, when the enemy has no weapon in hand.

**Notes** — The no-weapon case is guarded on purpose: dropping the shield whenever the player has nothing raised would let a player holster, watch the shield fall, and draw again. Requiring an actual reload means the shield opens exactly when the player cannot shoot anyway, which reads as the creature being smart rather than exploitable.

## `StaminaHit` — the anti-aim drain

**Contract** — Fired by the shared anti-aim ability each time it lands. Drains the player's stamina in proportion to the *weight of the weapon they are holding*, and knocks the weapon out of their hands if the drain would take them below a multiple of the drain itself. Does nothing in the developer invulnerability mode or when the player holds no weapon.

```text
FUNCTION StaminaHit()
  weapon = player's active weapon ; IF none RETURN
  drain  = weapon.weight * weight_to_stamina_hit
  drop   = player.stamina < drain * weapon_drop_stamina_k
  player.stamina_hit(drain)
  IF drop
     direction = player's facing, mirrored to point upward if it points down
     give the weapon a launch velocity of direction * weapon_drop_velocity
     ask the inventory to drop it; force the drop if the inventory refuses
```

**Notes** — Scaling by weapon weight makes the ability a counter to heavy weapons specifically: a player with a rifle is disarmed long before a player with a pistol. The upward mirroring of the drop direction is what makes the weapon arc away rather than being shoved into the floor. The forced fallback exists because the inventory can refuse a drop for reasons unrelated to this ability, and the design wants the disarm to be unconditional.

## `Die`

**Contract** — Cleans up both abilities on death: stops any running triple animation and releases every telekinetically held object, which drops them to the physics simulation.

## `net_Relcase`

**Contract** — The object-teardown notification. Drops the destroyed object from the telekinesis list, and deactivates the wave if that object was its victim.

**Invariants** — After this, no reference to the destroyed object survives in the creature. This is the creature-side half of an engine-wide contract; a rebuild whose references cannot dangle still needs the *semantic* effect — a destroyed victim ends the wave.

## `face_enemy`

**Contract** — The shared positioning step every burer attack sub-state calls: if not already aimed within about twenty degrees of the enemy, turn toward them; then stand idle. Does nothing when there is no enemy.

**Notes** — The tolerance means a burer does not micro-correct its facing, which matters because every one of its attacks plays an override animation that a turn would interrupt.

## `shedule_Update`

**Contract** — Runs the telekinesis ability's own coarse update alongside the base creature's. Telekinesis is advanced on the scheduler's cadence rather than per frame; the wave, which must not skip, is advanced per frame instead.

## `StartGraviPrepare` / `StopGraviPrepare`

**Contract** — Start and stop the wind-up effect drawn *on the victim*, slightly above their origin, for the duration of the gravity attack's charge. Only applies when the victim is the player. Stopping is unconditional on the player, not on the remembered victim.

## `StartTeleObjectParticle` / `StopTeleObjectParticle`

**Contract** — Mark or unmark a telekinetically held object with the hold effect, attached to the object itself.

## `reinit` / `~Burer` / `net_Destroy`

**Contract** — `reinit` lowers the shield and clears the scan timestamp. The constructor creates the state manager and the fast-gravity ability and registers the latter in the creature's first custom control slot; the destructor releases both. `net_Destroy` delegates.
