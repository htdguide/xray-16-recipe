# src/xrGame/ZoneCampfire.cpp

> A campfire: an anomaly zone that scripts can switch on and off, cross-fading its light, its particles and its sound over three seconds instead of popping.

**Needs** — [`ZoneCampfire.h`](ZoneCampfire.h.md) · [`MosquitoBald.h`](MosquitoBald.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame colour and range interpolation against a light and a particle emitter

## Purpose

A campfire is modelled as an anomaly zone rather than as scenery because it damages what
stands in it, and because the zone base already owns everything a campfire needs: an
effect volume, an idle particle emitter, an animated idle light and a scheduled update.
What this file adds is the only thing a campfire has that an anomaly does not — it can be
*extinguished and relit on command from a script*, and the transition has to look like a
fire dying rather than like a switch.

The whole file is therefore about one problem: a zone's enabled/disabled state is a
boolean that flips instantly, but a fire going out is a three-second fade. The solution is
a transition deadline that the state change sets and the update loop drains, during which
the light and the particles are driven off the remaining fraction rather than off the
boolean.

## State

```text
RECORD CampfireState
  enabling_particles  : optional<emitter>   # one-shot "catching light" burst, alive only during a turn-on
  disabled_particles  : optional<emitter>   # persistent smoulder/smoke shown while out
  disabled_sound      : optional<sound>     # looping sound of an extinguished fire, positioned at the zone
  turned_on           : bool                # target state; true at construction
  turn_deadline       : int                 # global-clock time the transition ends; 0 means "not transitioning"
```

Invariants:

- `turn_deadline == 0` exactly when no transition is in flight. It is the single flag the
  rest of the file branches on, which is why it is cleared rather than left stale.
- `disabled_particles` exists if and only if the zone is in the disabled state; entering
  the disabled state asserts it is currently absent, so a double-disable is a bug, not an
  idempotent no-op.
- The transition length is 3 seconds of wall clock, and two further thresholds inside it
  (see below) are expressed relative to that length, so changing it changes all three.

## `turn_on_script` / `turn_off_script`

**Contract** — the script-visible commands. Each records the target state, sets the
transition deadline three seconds ahead on the global clock, and immediately performs the
underlying zone state change; the *visual* consequences are then spread over the
transition by the update loop. Neither blocks.

**Invariants** — both are silently ignored unless the active render backend is of the
deferred generation or newer. The oldest forward renderer has no dynamic light model that
can be faded this way, so rather than produce a half-working effect the command does
nothing at all. This is a visible gameplay difference across render backends and a
rebuild that supports only one backend can drop the guard.

## `is_on`

**Contract** — the target state, not the displayed state: during a transition it already
reports the state being moved *to*.

## `GoEnabledState`

**Contract** — the zone-base hook for becoming active. Tears down every artefact of the
extinguished look — the smoulder emitter (stopped, then released) and the looping
extinguished sound — and starts the one-shot "catching light" burst named by the
configuration section, parented to the zone's transform so it follows it.

**Notes** — the burst is created with its own lifetime rather than being pooled, and is
destroyed later by the idle-particle handover (below), not here.

## `GoDisabledState`

**Contract** — the zone-base hook for going inert. Creates the smoulder emitter and the
looping extinguished sound from the section's `disabled_particles` and `disabled_sound`
keys, parents the emitter to the zone and plays the sound positioned at the zone with
looping on. Fails hard if a smoulder emitter already exists.

## `shedule_Update`

**Contract** — the scheduled (rate-degraded) update. Two jobs beyond the base's: keep the
transition draining even while the zone is disabled, and blow the idle smoke with the
weather system's wind.

```text
FUNCTION scheduled_update(elapsed)
  IF NOT enabled AND transition in flight THEN
    update_workload(elapsed)          # the base only runs the workload while enabled;
                                      # a fire going out still has to finish fading

  IF idle_particles exist THEN
    wind = environment.wind_direction scaled by environment.wind_strength
    idle_particles.reparent(transform, velocity: wind)

  base.scheduled_update(elapsed)
```

**Notes** — feeding the emitter a parent *velocity* rather than moving its particles is
what makes campfire smoke lean with the weather at no simulation cost; the particle system
integrates the inherited velocity itself.

## `UpdateWorkload`

**Contract** — the per-frame work the zone does while it matters. Runs the base's workload,
then, if a transition is in flight, drives the light and the particles off the fraction of
the transition remaining; when the deadline passes it clears the transition and commits
the final particle state. Allocates nothing.

**Invariants** — the interpolation parameter runs 0→1 over the transition when turning on
and 1→0 when turning off (the same remaining-fraction, complemented). The light's colour
*and* its range are both scaled by it, so a dying fire shrinks its pool of light as well
as dimming it — dimming alone reads as fog, not as extinction.

```text
FUNCTION update_workload(elapsed)
  base.update_workload(elapsed)

  IF turn_deadline > now THEN
    k = (turn_deadline - now) / TRANSITION_LENGTH     # 1 at the start, 0 at the end

    IF turning_on THEN
      k = 1 - k
      play_idle_particles(with_light: true)           # gated internally; see below
      start_idle_light()
    ELSE
      stop_idle_particles(with_light: false)          # gated internally

    IF idle_light is active THEN
      base_colour = idle_light_animation.sample(global_time)
      light.colour = base_colour scaled by k
      light.range  = (idle_light_range + random in [-0.25, +0.25]) scaled by k

  ELSE IF turn_deadline != 0 THEN
    turn_deadline = 0                                 # transition over: commit
    IF turned_on THEN play_idle_particles(with_light: true)
    ELSE stop_idle_particles(with_light: true)
```

**Notes**

- The light's colour comes from a named, looping light *animation* sampled on the global
  clock — the flicker of a fire is authored data, not noise generated here. The transition
  only scales it.
- The per-frame random jitter on the range (a quarter of a metre either way) is applied
  *before* the fade scale, so a fire that is nearly out flickers proportionally less. That
  ordering is the difference between a guttering flame and a strobing one.
- The colour channels are read back from the animation in blue-green-red order and written
  in red-green-blue order — the animation's packed form and the light's are opposite. A
  rebuild should pick one order and keep it; the swap here is an artifact of two different
  packed-colour conventions meeting, and is load-bearing only in that the shipped light
  animations were authored against it.

## `PlayIdleParticles`

**Contract** — starts the steady burning look, but only once the transition is at least
two thirds done, so the "catching light" burst has time to read as a separate event before
the steady flame replaces it. When it does fire, it also stops and releases the burst. If
no transition is in flight it passes straight through to the base.

## `StopIdleParticles`

**Contract** — stops the steady burning look, but only after the first sixth of the
transition, so that an extinguishing fire visibly persists for a moment rather than
vanishing on the command. Passes through when no transition is in flight.

**Notes** — the two thresholds (two seconds in for ignition, half a second in for
extinction) are asymmetric on purpose: catching light is slow, being put out is fast.
They are authored timings with no derivation beyond that.

## `AlwaysTheCrow`

**Contract** — while a transition is in flight the object demands an unconditional
per-frame update, overriding the scheduler's distance- and load-based rate degradation.
Otherwise it defers to the zone base. A three-second cross-fade updated at the coarse
scheduled rate would step visibly, so the fire buys itself full-rate updates for exactly
as long as it needs them and gives them back.
