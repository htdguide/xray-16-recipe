# src/xrGame/CustomZone.cpp

> The anomaly base class: a volume that notices what is inside it, cycles through idle, waking, blowout and recharge, hits everything in range on the blowout, and occasionally leaves an artefact behind.

**Needs** — [`CustomZone.h`](CustomZone.h.md) · [`space_restrictor.h`](space_restrictor.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`Artefact.h`](Artefact.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`zone_effector.h`](zone_effector.h.md) · [`Hit.h`](Hit.h.md) · [`BreakableObject.h`](BreakableObject.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a state machine over a contact set, with effect scheduling against a millisecond clock

## Purpose

Every anomaly in the game is this class plus an override of one method. The base owns
everything an anomaly has in common — the volume, the set of objects inside it, the state
cycle, every particle, light, sound and wind effect, the damage constructor, the artefact
economy — and leaves exactly one thing abstract: what the anomaly *does* to one object.
[`GraviZone.cpp`](GraviZone.cpp.md) and [`HairsZone.cpp`](HairsZone.cpp.md) are two
fillings of that hole.

The design decisions worth carrying into a rebuild:

**A zone is a restrictor first.** It inherits the volume machinery that also constrains
where creatures may walk, so an anomaly is automatically something the pathfinder avoids
and something the alife simulation knows about. The shape may be several spheres and boxes,
not one primitive, which is why distance is measured to the *nearest sub-shape* rather than
to a centre.

**Membership is event-driven.** Objects are not queried per frame; the touch sense reports
entry and exit, and the zone keeps a record per resident. That record is where per-object
state lives — how long you have been inside, whether the zone has given up on you, which
particle effects are attached to you.

**The state cycle is time-table driven.** Four durations, read from configuration, define
how long each phase lasts. A duration of -1 means "forever, until something else moves us
on", which is how idle is expressed. The zone advances on the *elapsed milliseconds in the
current state*, and effects within the blowout are scheduled by comparing that counter
against per-effect offsets — so a blowout can flash before it rumbles before it hurts.

**Two update rates.** Close to the camera the zone updates every frame; beyond fifty
metres it drops to the scheduler's rate. The switch is automatic and per-zone, and a level
full of anomalies therefore costs roughly what the few near ones cost.

**State changes go through the network.** A zone never changes its own state directly: it
sends an event to itself and applies the change when the event comes back. In single
player that is a round trip through the local event queue; in multiplayer it is what makes
every client's anomalies flash at the same moment.

## State

```text
ENUM ZoneState
  idle        # nothing active inside
  awaking     # something entered; winding up
  blowout     # the discharge
  accumulate  # recharging
  disabled    # switched off entirely

RECORD ZoneObjectInfo                   # one per resident
  object            : game object
  small_object      : bool      # radius below the small-object threshold (0.6)
  nonalive_object   : bool      # not a living creature, or a dead one
  zone_ignore       : bool      # the zone has stopped caring about this one
  particles         : list<particle effect>
  time_in_zone      : int       # milliseconds
  time_affected     : real      # wall clock of the last effect, for subclasses

RECORD Zone
  state              : ZoneState
  state_time         : int      # milliseconds in the current state
  previous_state_time: int      # the same, last tick: together they bracket this tick
  state_durations    : list<int> per state    # -1 means unbounded
  max_power          : real     # hit strength at the centre
  attenuation        : real     # falloff shape; see RelativePower
  effective_radius   : real     # fraction of the shape's radius the effect reaches
  hit_type           : damage type
  residents          : list<ZoneObjectInfo>
  spawned_artefacts  : list<Artefact>         # pre-spawned, parented to the zone, hidden
  artefact_table     : list<(section, normalized probability)>
  owner_id           : entity id              # who placed it, for a player-deployed mine
  ttl                : int                    # absolute expiry, only for a placed mine
  distance_to_viewer : real
  flags              : 20 configured booleans # see CustomZone.h
```

Invariants worth stating because nothing else states them:

- `previous_state_time` and `state_time` bracket the current tick; every scheduled effect
  fires exactly once because it tests whether its offset falls *inside* that half-open
  interval. This is how effects survive both a 16-millisecond frame and a 200-millisecond
  scheduled tick without firing twice or being skipped.
- every blowout effect offset is clamped at load time to the blowout's own duration, so an
  effect can never be scheduled past the end of the state it belongs to.
- a zone with residents but all of them ignored counts as *inactive* and returns to idle;
  the active flag is recomputed from scratch every scheduled tick.
- the artefact probability table is normalized at load, so it is a proper distribution
  regardless of what the author wrote, and a table summing to zero is a hard failure.

## `Load`

**Contract** — reads the entire anomaly description from one configuration section: the
four state durations, the three give-up timeouts, the hit type and impulse fraction, the
effective radius, the three "ignore" filters, a dozen optional particle names, six optional
sounds, the idle and blowout light descriptions, the wind profile, the artefact table, the
secondary-hit power, and the two numbers by which the AI evaluation layer classifies the
anomaly. Nearly every field is optional; an anomaly with no particles, no light and no
sound is legal and invisible.

Three parts of the load are more than reading:

```text
# 1. every blowout effect offset is clamped to the blowout duration
FOR EACH offset IN (particles, light, sound, explosion)
  IF offset > duration[blowout] THEN
    offset = duration[blowout]
    report a data error

# 2. the wind profile must be ordered, and must end inside the blowout
REQUIRE wind_start < wind_peak < wind_end
IF wind_end >= duration[blowout] THEN wind_end = duration[blowout] - 1

# 3. the artefact table is (section, weight) pairs, normalized into probabilities
weights must be an even-length list          # else a hard failure
total = sum of weights
REQUIRE total is non-zero
FOR EACH entry: entry.probability = entry.weight / total
```

**Notes**

- Idle's duration is *assigned* -1 rather than read: a zone stays idle until something
  enters it, by definition.
- The two "object idle particle" keys are read into the wrong fields — the small key's
  value lands in the big slot and the big key's value in the small slot, and each is read
  under the other's guard. The net effect is that both fields end up holding the value of
  a key named after the *other* size. This is a live defect and the shipped data was tuned
  around it; a rebuild that fixes it will change how existing anomalies look.
- An anomaly may attach a screen post-process effect, described in its own file, that
  intensifies as the player approaches. That is the visual signature of walking into an
  anomaly and it is entirely optional.

## `net_Spawn`

**Contract** — completes the zone from its authoritative record: starting power (the
section may override the record's), falloff, owner, the enable/disable duty cycle, and the
two render lights. Then switches the zone on, starts the idle presentation, and stamps the
movement tracking.

```text
FUNCTION spawn(record)
  base.spawn(record)
  max_power   = section's "max_start_power" IF present ELSE record.max_power
  attenuation = section's "attenuation"
  owner_id    = record.owner_id
  ttl         = now + 40 seconds IF the zone has an owner ELSE never

  IF this is a multiplayer game THEN disable artefact spawning

  # the authored duty cycle: t seconds on, t seconds off, offset by a start shift
  time_to_disable, time_to_enable, time_shift = record's, in milliseconds
  start_time = now
  use_duty_cycle = both durations are non-zero

  IF idle light is configured AND the renderer generation permits it THEN
    create a light, with shadows and volumetrics as configured
  IF blowout light is configured THEN create a shadowing light

  enable the object
  start idle particles, idle sound and idle light
  state_time = previous_state_time = 0
  record the position for movement tracking

  IF the spawn record's own overrides request it THEN force permanent fast mode
```

**Invariants** — a zone with an owner is a *deployed* object — a mine someone threw — and
expires after forty seconds. An authored anomaly has no owner and never expires. That
single field is the whole difference between level furniture and a thrown weapon.

**Notes** — the oldest renderer generation is given a separate opt-out for idle lights,
because a per-anomaly dynamic light was too expensive there. So the same level looks
different across render backends by design.

## `net_Destroy`

**Contract** — the teardown, and the order is load-bearing: stop the idle presentation
first (so particle objects release cleanly while the zone still exists), then the base,
then the wind (which must restore a global it borrowed), then the lights and the idle
particle object, then stop the screen effect, then run the exit hook for every remaining
resident and clear the set.

**Invariants** — every resident gets its exit hook. Residents carry attached particle
effects and, for the player, a depth-of-field override; skipping the exit would leak both.

## `shedule_Update` — the slow tick

**Contract** — the scheduler-rate update, which runs however far away the zone is. It does
the bookkeeping that must happen regardless of visibility: refresh the resident set from
the touch sense, age every resident and decide whether the zone has given up on it,
recompute whether the zone is active, trigger the wake-up, choose the update rate, and —
if the zone is in slow mode — run the workload here rather than per frame.

```text
FUNCTION scheduled_update(dt)
  active = false

  IF enabled THEN
    centre = the collision shape's sphere, in world space
    refresh the resident set against (centre, radius)      # touch sense

    FOR EACH resident
      resident.time_in_zone = resident.time_in_zone + dt

      # give-up rules, separately tuned for large and small objects;
      # a living creature is NEVER given up on
      timeout = give_up_small IF resident.small_object ELSE give_up_large
      IF timeout is set AND resident.time_in_zone > timeout
         AND resident is not a living creature THEN
        resident.zone_ignore = true

      IF idle_particle_timeout is set AND resident.time_in_zone > idle_particle_timeout
         AND resident is not a living creature THEN
        stop that resident's attached idle particles

      IF NOT resident.zone_ignore THEN active = true

    IF state is idle AND active THEN switch to awaking
    base.scheduled_update(dt)

    # choose the update rate
    IF distance from the camera to the shape's surface > 50 metres
       AND not forced fast THEN switch to slow ELSE switch to fast
    IF slow THEN run the workload with dt

  run the duty cycle
  IF multiplayer AND we own this object AND now > ttl THEN destroy self
```

**Notes** — the give-up rules are what stop a corpse in an acid pool from burning forever.
They deliberately exempt the living: a creature that stays in an anomaly keeps taking
damage until it dies, at which point it becomes non-alive and the timer starts to matter.

## `UpdateCL` — the fast tick

**Contract** — the per-frame update, and it runs the workload only while in fast mode. In
slow mode the per-frame processing is deactivated entirely, so this is not even called.

## `UpdateWorkload`

**Contract** — the actual state advance, called at either rate with that rate's delta. It
brackets the tick, runs the current state's handler, updates the idle light, recomputes
the distance to the viewer and feeds the screen effect, updates the blowout light, and
runs the secondary continuous damage if the anomaly has any.

```text
FUNCTION workload(dt)
  previous_state_time = state_time
  state_time = state_time + dt

  IF disabled THEN stop the screen effect; RETURN

  update the idle light's colour and flicker
  dispatch on state: idle | awaking | blowout | accumulate

  IF there is a viewer THEN
    # measured from a point slightly below the camera: roughly the player's chest
    distance, radius = distance to the nearest sub-shape from the camera, lowered 0.9 m
    feed the screen effect with (distance, radius, hit type)

  IF the blowout light is lit THEN fade it
  IF secondary hit is configured AND state is neither idle nor disabled THEN
    apply the continuous damage
```

**Notes** — the camera is lowered by nine tenths of a metre before measuring, so that the
screen effect's intensity tracks the player's *body* rather than their eyes. Without it,
crouching would change how strongly an anomaly affects you.

## The four state handlers

**Contract** — each answers whether it changed state. Idle runs the duty cycle and never
leaves on its own — `CheckForAwaking`, called from the scheduled tick, is what moves it.
The other three are pure timers:

```text
awaking    : when state_time reaches its duration -> blowout
blowout    : when state_time reaches its duration -> accumulate,
             and if the zone is "blowout once", disable it permanently
accumulate : when state_time reaches its duration ->
               blowout again IF something active is still inside
               ELSE idle
```

**Invariants** — the accumulate-to-blowout edge is what makes an anomaly pulse repeatedly
while you stand in it, and the accumulate-to-idle edge is what lets it go quiet once you
leave. Nothing else decides this.

## `UpdateBlowout`

**Contract** — the blowout's effect schedule, called by a subclass's blowout handler. Each
of five effects fires on the single tick whose bracket contains its configured offset:
particles, light, sound, wind start, and the damage pass. Wind is then updated every tick
while active.

```text
FUNCTION update_blowout()
  IF particles_offset falls in [previous_state_time, state_time) THEN play blowout particles
  IF light_offset     falls in that interval                     THEN start the blowout light
  IF sound_offset     falls in that interval                     THEN play the blowout sound
  IF wind is configured AND wind_start falls in it               THEN start the wind
  update the wind
  IF explosion_offset falls in that interval THEN
    affect every resident
    try to bear an artefact
```

**Invariants** — the damage pass and the artefact are on the *same* offset, so an anomaly
produces an artefact at the instant it hurts you, not before or after.

## `AffectObjects`

**Contract** — runs the subclass's per-object effect over every non-destroyed resident, at
most once per rendered frame however many times it is called, and never while the level is
still streaming in. The once-per-frame guard exists because both update paths can reach
it in the same frame during a rate switch.

## `Affect`

**Contract** — the one abstract operation: what this anomaly does to one resident. The base
does nothing. Everything else in the file exists to decide *when* this is called and on
whom.

## `RelativePower` / `Power` / `effective_radius`

**Contract** — the falloff. Strength is one at the centre and falls quadratically with
distance, scaled by the attenuation constant, clamped at zero, and zero outright beyond
the effective radius. The effective radius is a configured fraction of the *nearest
sub-shape's* radius, not of the whole zone — which is what makes a multi-lobed anomaly
behave as several small anomalies rather than one big one.

```text
FUNCTION relative_power(distance, nearest_shape_radius) -> real
  radius = nearest_shape_radius * effective_radius
  IF distance > radius THEN RETURN 0
  RETURN max(0, 1 - attenuation * (distance / radius)^2)

FUNCTION power(distance, nearest_shape_radius) -> real
  RETURN max_power * relative_power(distance, nearest_shape_radius)
```

**Notes** — with attenuation at 1 the strength reaches zero exactly at the radius; above 1
it reaches zero early, leaving a dead outer shell; below 1 it is still finite at the rim
and cuts off abruptly. All three are used by shipped anomalies.

## `CalcDistanceTo`

**Contract** — distance from a point to the zone, expressed as a pair: the distance to the
nearest sub-shape's centre, and that sub-shape's radius. A single-shape zone short-circuits
to the bounding sphere. A box sub-shape reports its largest half-extent as its radius.

**Invariants** — every caller uses the pair together, because the falloff is relative to
the sub-shape that is actually nearest. A rebuild that returns only a distance will get
multi-lobed anomalies wrong.

## `feel_touch_contact` / `feel_touch_on_contact`

**Contract** — the admission test. An object is admitted only if it is not itself a zone,
not a breakable prop, has an animated visual, is not this zone, declares itself visible to
zones, genuinely overlaps the collision shape, and agrees to the contact. The last step is
mutual consent: the candidate is asked whether it accepts this zone, so an object can
opt out of specific anomalies.

**Notes** — excluding zones from each other is what stops two overlapping anomalies from
entering a feedback loop. Excluding breakables is unexplained; they are presumably too
numerous.

## `feel_touch_new`

**Contract** — classifies a newly admitted object and decides whether the zone cares about
it at all. Size and aliveness are determined once, at entry; the three "ignore" filters are
applied against them; and the resident is recorded, the entry hook run, and the entrance
and per-object idle effects started.

```text
FUNCTION on_enter(object)
  info.object          = object
  info.nonalive_object = NOT (object is a living creature AND alive)
  info.small_object    = object.radius < 0.6
  info.zone_ignore = (small AND ignore_small)
                  OR (nonalive AND ignore_nonalive)
                  OR (object is an artefact AND ignore_artefacts)
  enter_zone(info)
  residents.append(info)
  IF enabled THEN
    play entrance effects for this object
    start per-object idle effects on this object
```

**Invariants** — size and aliveness are *frozen at entry*. A creature that dies inside an
anomaly is still classified as alive for as long as it stays, which is exactly why the
give-up timers re-check aliveness every tick rather than trusting the flag.

## `feel_touch_delete` / `net_Relcase`

**Contract** — the two ways a resident leaves: walking out, and being destroyed. Both run
the exit hook and remove the record. The destruction path additionally forgets the owner if
the destroyed object was it, and stops the screen effect if it belonged to the destroyed
object. This is the conformance invariant — a destroyed entity must be unreferenced by
every subsystem — and a zone holds three separate references to worry about.

**Notes** — the walk-out path skips stopping the per-object particles if the object is
already being destroyed, because the object's own teardown will handle them.

## `enter_Zone` / `exit_Zone`

**Contract** — the per-resident hooks. The base's only behaviour is the player's
depth-of-field override: an anomaly may declare that being inside it blurs the view, and
the flag is set on the player's entry and cleared on their exit. Subclasses extend these.

## `SwitchZoneState` / `OnStateSwitch`

**Contract** — the split that makes state changes networked. The *request* side, run only
on the authoritative side, emits a state-change event and resets the tick bracket. The
*apply* side, run when that event arrives, enables or disables the zone, plays the state's
entry effects, and commits the new state.

```text
FUNCTION request_state(new_state)                  # authoritative side only
  send a zone-state-change event carrying new_state
  state_time = previous_state_time = 0

FUNCTION apply_state(new_state)                    # on receiving that event
  IF new_state is disabled THEN disable() ELSE enable()
  IF new_state is accumulate THEN play accumulate effects
  IF new_state is awaking    THEN play awaking effects
  state = new_state
  state_time = previous_state_time = 0
```

**Invariants** — exactly one state-change event may be in flight per state, which the
original notes and does not enforce. The requesting side resets its own clock immediately
rather than waiting for the event, so the two sides' clocks agree even though the state
does not change until the event lands.

## `Enable` / `Disable`

**Contract** — the presentation halves of the state change. Enabling forces fast mode and
restarts the entrance and idle effects on every current resident plus the zone's own idle
presentation. Disabling forces slow mode, stops every per-object effect, stops the idle
presentation and stops the screen effect. Each is a no-op if already in that condition.

**Notes** — enabling returns true and disabling returns false, unconditionally. The values
look like they were meant to mean "did something change" and do not.

## `UpdateOnOffState` / `GoEnabledState` / `GoDisabledState`

**Contract** — the duty cycle: an authored anomaly may be configured to switch itself on
and off on a fixed period, phase-shifted by a per-instance offset so that a field of them
does not pulse in unison. The desired condition is computed from the wall clock modulo the
period, and a mismatch with the current state triggers a state-change event.

```text
FUNCTION duty_cycle()
  IF no duty cycle configured THEN RETURN
  t = (now - start_time + phase_shift) mod (on_duration + off_duration)
  want_on = t < on_duration
  IF state is disabled AND want_on      THEN request enabled
  IF state is idle     AND NOT want_on  THEN request disabled
```

**Invariants** — going disabled *evicts every resident*: the exit hook runs for each, the
resident set is emptied and the touch sense's membership is cleared. A zone that switches
back on rediscovers whoever is standing in it. This is what makes a periodic anomaly safe
to walk through between pulses.

**Notes** — the transition is only checked from disabled and from idle, so a zone
mid-blowout finishes its blowout before the duty cycle can switch it off.

## `BornArtefact` / `SpawnArtefact` / `PrefetchArtefacts` / `ThrowOutArtefact`

**Contract** — the artefact economy, and it is deliberately indirect. Artefacts are
*pre-spawned* — created ahead of time, parented to the zone, made invisible and disabled —
so that the moment of birth costs nothing. A blowout rolls against the spawn probability
and, on success, releases one of the held artefacts by emitting an ownership-reject event;
when that event returns, the artefact is unparented and thrown.

```text
FUNCTION bear_artefact()                 # on the blowout's damage tick
  IF artefact spawning is off OR nothing is held THEN RETURN
  IF random(0,1) > spawn_probability THEN RETURN
  top up the held pool to its size (1)
  artefact = take one from the pool
  IF we are authoritative THEN emit an ownership-reject event for it

FUNCTION spawn_one()                     # topping up the pool
  pick a section from the artefact table by its normalized probability
  spawn that section at the zone's centre, owned by this zone

FUNCTION throw_out(artefact)             # when the reject event returns
  place it at the zone's centre, raised by the configured spawn height
  play the birth particles and sound
  apply an impulse in a uniformly random direction at the configured strength
```

**Invariants** — the held pool is exactly one artefact deep, topped up at the moment of
use rather than kept full, so a zone that never fires never holds anything. The random
throw direction is fully three-dimensional, so an artefact can be thrown downward into the
ground; nothing prevents it.

**Notes**

- Artefact spawning is disabled outright in multiplayer at spawn time, because artefacts
  there are match objects placed by the game mode.
- The birth path goes through the ownership events rather than calling directly, so that
  clients see the artefact appear at the same time the server does. In single player it is
  a local round trip that could be collapsed.

## `OnEvent`

**Contract** — three events. A state change applies the new state. An ownership-take parents
an artefact to the zone and hides it, which is how a pre-spawned artefact joins the pool.
An ownership-reject unparents one and, unless the rejection is part of the artefact's own
destruction, throws it.

**Invariants** — the reject carries a flag distinguishing "released into the world" from
"released because it is being destroyed". Throwing a dying artefact would apply an impulse
to a shell that is about to be freed.

## `CreateHit`

**Contract** — the zone's damage constructor. Builds a hit record naming the zone as the
weapon and, when the zone was deployed by someone, that someone as the attacker, and emits
it as an event. Runs only on the authoritative side.

**Invariants** — attributing the hit to the deploying player rather than to the mine is
what makes a player-placed anomaly score a kill for its owner.

## The presentation methods

**Contract** — a dozen small methods, each starting one visual or audible effect. Their
shared decisions:

- **Size branching.** Entrance, hit and per-object idle effects each have a "big" and a
  "small" variant, chosen by the object's radius against the small-object threshold. So a
  bolt entering an anomaly produces a different splash from a body entering it.
- **Bone attachment.** Object-attached effects pick a random bone of the target and play
  there, oriented along the object's velocity when it has one and upward when it does not.
  A tumbling body therefore sparks at a different place each time.
- **Effect ownership.** Every object-attached effect is tagged with the zone's identifier,
  so an object inside two anomalies has two independent effect sets and one zone stopping
  its effects does not cancel the other's.
- **The idle light flickers** by sampling a named light-animation curve for its colour and
  jittering its range by a configured delta each frame. The curve is authored data shared
  with the rest of the engine's light animation.
- **The blowout light fades** over its configured lifetime, scaling both colour and range
  by the remaining fraction raised to the power 0.15 — an extremely flat curve, so the
  flash holds near full brightness for most of its life and collapses at the end. A linear
  fade reads as a dimming lamp rather than as a flash.

## `PlayBoltEntranceParticles`

**Contract** — the special case: an anomaly may declare a distinct effect for a *bolt*
thrown into it, which is the player's tool for locating anomalies. Rather than one effect
at the bolt, it scatters effects across the whole volume — for each spherical sub-shape, a
number of instances proportional to its radius, each at a pseudo-randomly spread direction
and distance from the centre. The result reveals the anomaly's extent, which is the entire
point of throwing a bolt.

**Notes** — box sub-shapes are skipped, so a box-shaped anomaly reveals nothing. The
angular spread multiplies a random value by the loop index, so successive instances are
spread further apart — a cheap spiral rather than a uniform distribution.

## `StartWind` / `UpdateWind` / `StopWind`

**Contract** — a blowout may drive the *global* weather wind, ramping it from the ambient
value up to a configured peak and back down again over the blowout, but only while the
player is within four zone radii. The previous global value is saved on start and restored
on stop.

```text
FUNCTION update_wind()
  IF not active THEN RETURN
  IF viewer is beyond 4 * radius OR state_time is past wind_end THEN stop; RETURN
  IF state_time < wind_peak THEN
    interpolate from the saved ambient value up to the peak
  ELSE
    interpolate from the peak back down to the saved ambient value
  clamp into [0, 1]
```

**Invariants** — the saved value must be restored exactly once. Teardown restores it
unconditionally, because a zone destroyed mid-blowout would otherwise leave the weather
permanently windy.

**Notes** — this is a game object reaching into a global environment parameter, which is
the service-locator cycle that
[`SYSTEM-REQUIREMENTS.md` §7](../../SYSTEM-REQUIREMENTS.md#7-build-order) warns about. Two
anomalies blowing out at once will fight over it, and the last to stop restores whatever
it saved — which may be the other's peak.

## `o_switch_2_fast` / `o_switch_2_slow` / `AlwaysTheCrow`

**Contract** — the update-rate switch. Fast mode registers per-frame processing and lights
the idle light; slow mode deregisters it and, unless the zone opts out, extinguishes the
idle light. `AlwaysTheCrow` tells the scheduler this object must not be skipped: true
whenever the zone is doing anything other than idling or sitting disabled.

**Invariants** — extinguishing the idle light in slow mode is a visible trade: a distant
anomaly stops glowing. The opt-out exists precisely because that was noticed, and the base
opts out by default.

## `Hit`

**Contract** — a zone hit by a bullet plays a small entrance effect at the impact point and
takes no damage. Anomalies cannot be destroyed by shooting them; the effect exists so that
shooting one tells you it is there.

## `OnMove`

**Contract** — for anomalies that move: derives a velocity from the position change since
the last call and feeds it to the idle particle system, and repositions both lights.
Particle systems need the velocity so that emitted particles inherit the anomaly's motion
rather than hanging in the air behind it.

## `save` / `load`

**Contract** — the only thing persisted is one byte of state, and it is deliberately lossy:
a zone saved in any state other than disabled is restored as *idle*. So reloading a save
never catches an anomaly mid-blowout. Everything else — residents, timers, held artefacts —
is rebuilt from the world.

**Notes** — this is the right call and a rebuild should copy it: mid-blowout state would
have to be reconciled against a resident set that no longer exists.

## `net_Import` / `net_Export`

**Contract** — pure delegations. A zone's replicated state is entirely its base restrictor's;
the state machine is driven by events instead.

## Could not recover

- `m_fSecondaryHitPower` is loaded when secondary hits are enabled and the method that
  would apply it is empty in the base. No subclass in this slice overrides it, so the
  "fire and poison mines deal continuous small damage" feature is configured and inert
  here.
- `time_affected` in the resident record is stamped at entry and never read by the base.
- The small/big object idle particle keys are read into swapped fields (see `Load`).
- `Enable` and `Disable` return constants rather than whether anything changed.
- The prefetch depth is a compiled-in one artefact; nothing explains why a pool exists at
  all rather than spawning on demand, unless the spawn cost was once higher.
