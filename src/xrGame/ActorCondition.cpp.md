# src/xrGame/ActorCondition.cpp

> The player's body as a set of slowly-moving numbers: stamina spent by moving, hunger, alcohol, radiation, psychic health, temporary boosts, and the thresholds at which the player starts to limp, cannot run, or dies.

**Needs** — [`ActorCondition.h`](ActorCondition.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level.h`](Level.h.md) · [`Wound.h`](Wound.h.md) · [`Weapon.h`](Weapon.h.md) · [`PDA.h`](PDA.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`autosave_manager.h`](autosave_manager.h.md) · [`ai/monsters/basemonster/base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`ui/UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`ActorCondition.h`](ActorCondition.h.md); callers name that, not this file.
**Tier floor** — T3: scalar integration over time plus threshold bookkeeping

## Purpose

Every creature has a condition — health, stamina, bleeding, radiation — described in
[`EntityCondition.cpp`](EntityCondition.cpp.md). The actor's is different in kind, not
degree: it is the game's difficulty model. Hunger, alcohol, psychic health, per-anomaly
damage ceilings, consumable boosts and the weight the player can carry all live here,
along with the hysteresis bands that decide whether the player limps.

None of these numbers is authored per level. They are read from one configuration section
and integrated continuously, so the whole of survival in this game is a handful of rates
in an `ltx` file. That is the reason to keep this file separate: it is where the game's
pacing is tuned.

## State

```text
RECORD ActorCondition                      # extends the shared entity condition
  alcohol            : real   # 0..1; decays toward 0 by a configured rate (which is NEGATIVE in data)
  satiety            : real   # 0..1; hunger. 1 is full. Drains at a fixed rate.
  satiety_critical   : real   # the point below which hunger HARMS instead of healing
  v_satiety          : real   # drain rate per second
  v_satiety_health   : real   # health change per second, scaled by the satiety coefficient
  v_satiety_power    : real   # stamina regeneration per second, scaled by satiety

  power_leak_speed   : real   # maximum stamina ceiling falls continuously — see Notes
  jump_power, stand_power, walk_power        : real   # stamina cost per jump / per second
  jump_weight_power, walk_weight_power       : real   # extra cost proportional to carried weight
  overweight_walk_k, overweight_jump_k       : real   # multiplier once over the limit
  accel_k, sprint_k                          : real   # cost multipliers when running / sprinting
  max_walk_weight                            : real   # above this the actor cannot walk at all

  zone_max_power[5]  : real   # per-influence damage ceiling: radiation, fire, acid, psi, electric
                              # invariant: never zero — it is a DIVISOR in the interface
  zone_danger[5]     : real   # 0..1 per influence; what the detector and the interface display
  max_power_restore_speed, max_wound_protection, max_fire_wound_protection : real

  limping_power_begin/end, limping_health_begin/end   : real   # hysteresis bands
  cant_walk_power_begin/end, cant_sprint_power_begin/end : real
                              # invariant: begin <= end for every band, asserted at load

  condition_flags    : bitset # one latch per tutorial threshold already announced
  booster_influences : map<boost_kind, Booster>       # at most one live boost per kind
  current_medicine   : MedicineInfluence              # a timed multi-parameter effect
  death_effector     : optional<DeathEffector>
  time_affected      : real   # the ambient-damage accumulator's last consumed timestamp
```

**The hysteresis invariant is the important one.** Limping, cannot-walk and cannot-sprint
are each latched between two thresholds: the state turns *on* below the lower and *off*
only above the upper. Without the gap, a player resting exactly at the threshold flickers
between walking and limping every frame. A rebuild that collapses each pair to one number
will reproduce that flicker.

**The zone-ceiling invariant** — each of the five ceilings is asserted non-zero at load
because the interface divides an incoming hit by it to draw a 0..1 bar. It is a scale, not
a clamp: a ceiling of 2 means the bar reads half.

## `LoadCondition`

**Contract** — reads the whole table above from a section. The section may *redirect*: a
key names another section to read the condition from, so several entity types can share
one tuning block. Required keys fail loudly; the five zone ceilings and the three maxima
are optional and default to one.

**Notes** — the redirection key is read from the *entity's* section, and the fallback is
the entity section itself. That two-level lookup is what lets a modification retune the
player without editing the actor's own section.

## `UpdateCondition`

**Contract** — the per-update integration. Runs in game time scaled by the frame delta the
base class already computed. Does nothing for a dead actor, and nothing for a remote actor
that is not the one being viewed — a client does not integrate another player's hunger.

```text
FUNCTION update_condition()
  IF real-time god mode THEN
    advance satiety and boosts only; accumulate alcohol; drop the drunk camera effector
    # the real-time variant keeps the simulation running but refuses all harm
  IF god mode THEN RETURN
  IF NOT alive THEN RETURN
  IF not locally controlled AND not the viewed entity THEN RETURN

  weight_ratio = carried_weight / carry_limit
  IF moving THEN spend stamina for walking(weight_ratio, accelerated, sprinting)
  ELSE        spend stamina for standing(weight_ratio)

  IF single player THEN
    # The stamina CEILING itself falls, faster the more you carry.
    k = 1 + min(carried, limit)/limit + max(0, (carried - limit)/10)
    max_power -= power_leak_speed * dt * k

  alcohol += v_alcohol * dt, clamped to 0..1
  IF single player THEN
    attach or detach the drunk camera effector as alcohol crosses ~0
    attach or detach the psychic post-process effector as psy health leaves ~1,
      preferring a per-level variant of the effector section when one exists

  update_satiety(); update_boosters()
  run the shared entity condition update          # health, bleeding, radiation, wounds
  IF single player THEN check_tutorial_thresholds()

  IF health < 0.05 AND no death effector yet AND single player THEN
    start the death effector, if the data defines one
  IF a death effector is running THEN advance it, and stop it once it reports finished

  apply_ambient_damage()
```

**Invariants** — the base class's update runs *after* this class's own contributions have
been accumulated into the pending health and stamina deltas, because the base class is
what applies and clamps them. Reordering silently loses a frame of hunger.

**Notes**

- The falling stamina ceiling is the mechanic that makes long journeys tiring: resting
  restores stamina to a ceiling that is itself lower than it was an hour ago, and only
  sleeping or eating raises it back.
- The psychic post-process effector is looked up by a name built from the level's name,
  falling back to a generic one. That is the only place in the condition system where a
  *level* changes a rule, and it exists so that one level can have its own madness look.
- God mode has **two** forms and they differ: the plain one short-circuits everything
  including hunger, the real-time one keeps the simulation advancing and only refuses
  damage, which is what a spectating or debugging player wants.

## `UpdateSatiety`

**Contract** — drains hunger and converts it into health and stamina. In multiplayer,
hunger does not exist: the call degenerates to a flat stamina regeneration.

```text
FUNCTION update_satiety()
  IF not single player THEN pending_power += v_satiety_power * dt; RETURN

  satiety -= v_satiety * dt, clamped to 0..1

  # One coefficient, signed: positive above the critical point, negative below,
  # and normalized so it reaches ±1 at full and at empty.
  IF satiety >= critical THEN k = (satiety - critical) / (1 - critical)
  ELSE                        k = (satiety - critical) / critical

  IF the actor can be harmed THEN
    pending_health += v_satiety_health * k * dt      # heals when fed, harms when starving
    pending_power  += v_satiety_power * satiety * dt # stamina regen scales with fullness
```

**Notes** — the single signed coefficient is the whole design of hunger in this game: one
number, one authored critical point, and being well-fed is a slow regeneration rather than
a buff. Stamina regeneration uses raw satiety rather than the signed coefficient, so
starving stops regeneration but never drains stamina directly.

## `ConditionWalk` · `ConditionStand` · `ConditionJump`

**Contract** — the three ways movement costs stamina. Each takes the carried weight as a
*ratio* of the carry limit, so the cost formula is scale-free.

```text
walk:  cost = (walk_power + walk_weight_power * ratio * (ratio > 1 ? overweight_k : 1))
              * dt * (sprinting ? sprint_k : accelerated ? accel_k : 1)
jump:  cost =  jump_power + jump_weight_power * ratio * (ratio > 1 ? overweight_k : 1)
stand: cost =  stand_power * dt
```

**Invariants** — walking and jumping costs pass through the outfit's stamina-protection
factor before being subtracted; standing does *not*. A rebuild that routes standing
through the outfit too will make heavy armour free to stand in, which it is not meant to
be.

**Notes** — the jump cost is not multiplied by the frame time, because a jump is an event
rather than a duration. That asymmetry is correct and easy to get wrong.

## `IsLimping` · `IsCantWalk` · `IsCantSprint` · `IsCantWalkWeight`

**Contract** — the four movement restrictions. The first three are the latched hysteresis
bands over stamina (and, for limping, also over health: *either* being low starts the limp
and *both* must recover to end it). The fourth is not latched at all — it compares carried
weight against the walking limit directly, and also sets a flag the tutorial-threshold
check reads.

**Notes** — the three latches are mutable state read through const queries, which is the
C++ way of saying these are *observations that mutate*. A rebuild should make the latch
update part of the per-frame update and the queries pure; the behaviour is identical as
long as the update runs before any reader.

## `AffectDamage_InjuriousMaterialAndMonstersInfluence`

**Contract** — applies the continuous ambient damage the actor takes from standing on a
harmful surface and from being near psychically or radioactively active creatures. Runs on
a fixed tenth-of-a-second accumulator rather than per frame, and *catches up* by emitting
one tick per elapsed tenth-second, bounded to three ticks of backlog.

```text
FUNCTION apply_ambient_damage()
  TICK = 0.1 s
  IF now < last_tick + TICK THEN RETURN
  clamp last_tick to at least now - 3*TICK        # bound the catch-up burst

  radiation = harmfulness of the material underfoot
  FOR EACH creature the actor's detector currently feels AND that is alive
    radiation += its radiation aura
    psi       += its psychic aura
    fire      += its heat aura

  WHILE last_tick + TICK < now
    last_tick += TICK
    FOR EACH of (radiation, psi, fire) that is non-zero
      send a hit event against the actor, of the matching damage type,
      with magnitude aura * TICK, from straight above, with no bone and no impulse
```

**Invariants** — the damage is delivered as a **hit event through the network path**, not
as a direct field change, even in single player. That is the rule the whole chapter
follows: damage has one entry point, so that armour, wounds, sounds, script callbacks and
the multiplayer server all see it. The backlog bound is what stops a level-load hitch from
killing the player with three seconds of accumulated radiation in one frame.

**Notes** — the creature auras are gathered through the *detector item's* touch sense, not
the actor's. The player only takes aura damage from creatures the detector can feel, which
means the detector's own feel radius is a gameplay parameter of ambient damage. That is
almost certainly not what it looks like in the data.

## boosts

**Contract** — a boost is (kind, value, remaining time). At most one boost of each kind is
live; applying a second of the same kind cancels the first and replaces it. Applying adds
the value to the corresponding rate or immunity; expiry subtracts exactly the same value
back. Each kind maps to one field, and the two mappings — apply and remove — are the same
table with the sign flipped.

**Invariants** — apply and remove **must** be exact inverses, because the underlying fields
are shared with the outfit and the base condition. A boost that is applied twice and
removed once permanently changes the player's immunities for the rest of the save. The
replace-on-reapply rule exists precisely to make double application impossible.

**Invariant** — boosts are applied **only on the authoritative side**. A client applies
nothing; it receives the resulting values. The countdown, however, runs everywhere, and
runs in *game* time in single player (so that sleeping or accelerating time expires
boosts) and in real time in multiplayer.

**Notes**

- `ClearAllBoosters` removes every boost's effect but **does not empty the map**. The next
  update therefore counts down entries whose effects are already gone and removes them a
  second time — a double subtraction. This looks like a defect; it is reachable from
  script, so a rebuild should clear the map too and note the behavioural difference.
- Raising the carry limit is the one boost with a second effect: it raises both the
  inventory's maximum weight and the walking-weight limit by the same amount.

## `save` · `load`

**Contract** — serializes alcohol, the threshold latches, satiety, all nine fields of the
current medicine influence, and the live boost list as a count followed by
(kind, value, remaining) triples. Loading re-applies each boost's effect as it reads it,
which is how a saved boost resumes rather than merely counting down.

**Invariants** — the boost count is written as a single byte, capping live boosts at 255;
there are seventeen kinds, so this can never overflow. The field order is the frozen part —
this is the actor's contribution to the save format.

## `ApplyInfluence` · `ApplyBooster`

**Contract** — the two ways a consumable changes the actor. An *influence* is a timed
bundle of parameter changes and only one can be in progress at a time: a second is
refused outright, which is why eating a second bandage while one is working does nothing.
A *booster* is a single parameter for a duration and always succeeds, replacing any live
boost of its kind. An influence with a negative total duration is instantaneous and is
handed to the base class instead.

Both play the item's configured use sound, but only when the actor is both locally
controlled and the one being viewed — otherwise every player would hear every other
player eat. The sound is 2D and explicitly ignores the game's time factor, so it plays at
normal speed even during accelerated time.

## `UpdateTutorialThresholds`

**Contract** — fires a script callback the *first* time each of eight conditions is
reached: low stamina, low stamina ceiling, heavy bleeding, hunger, radiation, low psychic
health, overloaded, and a jammed weapon in hand. Each has a latch bit, so each fires once
per game. At most one callback fires per update — the checks are in a fixed priority
order and the first to trip suppresses the rest for that update.

**Invariants** — the callback is a global script function looked up by name and asserted
to exist. The eight names are part of the frozen script surface (criterion 10). The
thresholds are read once from a dedicated configuration section and cached for the
process's lifetime, which means changing them requires a restart.

**Notes** — firing at most one per update is deliberate throttling, not an oversight: the
callbacks show tutorial messages, and two at once would overlap on screen.

## `PlayHitSound` · `DisableSprint` · `HitSlowmo`

**Contract** — three per-damage-type policies, each a table:

- **sound** — psychic damage is always silent; impact and wound damage always sound; the
  four *field* types (radiation, burn, light burn, chemical) sound only above a small
  damage threshold, so that standing in a weak anomaly does not produce a continuous
  grunt.
- **sprint** — every damage type interrupts a sprint except the five that represent
  ambient fields, which the player must be able to sprint out of.
- **slow motion** — only bullet wounds and impacts produce the hit-stagger effect, scaled
  by damage and capped at one.

## `GetZoneMaxPower`

**Contract** — maps either an influence kind or a damage type to its ceiling. Out-of-range
influences return one. The damage-type mapping folds burn and light burn onto the fire
ceiling, and returns one for every purely physical type; ordinary bullet wounds return the
separate wound-protection maximum instead.

## `SetZoneDanger` · `GetZoneDanger`

**Contract** — anomalies report how dangerous they currently are, per influence kind,
clamped to 0..1. The aggregate is their sum, capped at 1.5 — deliberately above one, so
that standing in two anomalies at once reads as worse than standing in one at full
strength, without being unbounded. The sum skips index zero, which is a non-influence
placeholder.

## `CActorDeathEffector`

**Contract** — the death sequence. Constructed when health falls below a small threshold;
it blocks all weapons, hides the interface indicators, disables input, installs a
post-process effector and plays a 2D sound, and then **holds the actor's health at the
value it had at construction** every update. The actor is therefore not dead while the
effect plays. When the post-process effector finishes it sets health to a negative value,
which is what actually kills the actor, and the owner then tears the sequence down:
effector removed, sound destroyed, input and indicators restored.

**Invariants** — the health pin is the whole trick. Any other code that would kill the
actor during the sequence is overwritten each update, so the death animation always plays
to completion. The final health is set *negative* rather than zero, because zero is a
reachable living state elsewhere in the condition system.

**Notes** — a stray debug print remains in the release path of the effector-released
callback. A rebuild should drop it.
