# src/xrGame/CarSound.cpp

> The car's engine-audio state machine — a small four-state automaton that turns
> the vehicle's mechanical state into a start sample, a pitched loop and a stop sample.

**Needs** — [`Car.h`](Car.h.md) · [`Hit.h`](Hit.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it owns sound handles and reads a clock, but nothing here needs manual
memory or a fixed byte layout.

## Purpose

A running engine is not one sound. It is a start transient, a loop whose pitch tracks
revolutions, and a stop transient — and the transitions between them must not overlap or
cut each other off. This file is the automaton that gets that right, kept separate from the
rest of the vehicle because it is the only part of a car that cares about wall-clock time
rather than the physics step.

It is a nested part of the car, not a class of its own: the sound state lives inside the
car record and reaches back into the car for the transform and the current revolutions.

## State

```text
ENUM CarSoundState = off | starting | drive | stalling | stopping

RECORD CarSound
  car                 : Car             # owner; read for transform and revolutions
  state               : CarSoundState
  volume              : real            # from the model's own configuration, default 1
  relative_pos        : vector          # emitter offset in car space, default (0, 0.5, -1)
  time_state_start    : int             # clock reading when the current state was entered
  engine_start_delay  : int (ms)        # see below
  snd_engine          : sound           # the loop
  snd_engine_start    : sound           # the start transient
  snd_engine_stop     : sound           # the stop transient, also used for stalling
  snd_transmission    : sound           # optional gear-change click
```

**Invariants** — while `state` is anything but `off` the car is registered with the
per-frame update scheduler; entering `off` unregisters it. That pairing is the whole reason
`switch_on`/`switch_off` exist as separate operations from setting the state. The emitter
position is recomputed from the car's transform every frame for whichever sample is
currently the live one, so a sound never lags the vehicle.

Every entry point asserts the physics world is **not** mid-step. Audio is driven from the
frame update, never from inside a physics callback, because the car's transform is only
coherent between steps.

## configuration

**Contract** — the car's tuning lives in the *model's* embedded configuration (the user
data block carried inside the visual), not in the entity's ltx section. A car whose model
has no `car_sound` section is not an error: it logs a complaint and stays silent forever.

```text
FUNCTION init(car)
  ini = user data of car's visual
  IF ini has no section "car_sound" OR no volume key
      log "car has no sound params"; state = off; RETURN
  volume       = ini.car_sound.snd_volume
  snd_engine   = load(ini.car_sound.snd_name)             # required once the section exists
  start        = load(ini.car_sound.engine_start  else a default start sample)
  stop         = load(ini.car_sound.engine_stop   else a default stop  sample)
  fraction     = ini.car_sound.engine_sound_start_dellay else 0.25
  engine_start_delay = floor(length_of(start) in ms * fraction)
  relative_pos = ini.car_sound.relative_pos       if present
  snd_transmission = load(ini.car_sound.transmission_switch) if present
  state = off
```

**Notes** — `engine_start_delay` is a *fraction of the start sample's own length*, not a
fixed time. The loop is meant to fade in partway through the cranking sound, and where that
moment falls depends on how long the sample is; expressing it as a fraction lets one number
serve every vehicle. The default quarter-way point is the value the shipped data relies on.

The default start and stop samples are hard-coded fallbacks so that a mod-added vehicle that
only names a loop still makes a plausible noise.

## the state machine

**Contract** — five commands drive the automaton, and one per-frame update advances it. All
of them are no-ops in `off` except `start` and `drive`, which switch it on.

```text
FUNCTION start()            # ignition
  IF state == off THEN switch_on()          # registers with the scheduler
  enter(starting); play(snd_engine_start, one-shot); place(snd_engine_start)

FUNCTION drive()            # engine running under load
  IF state == off THEN switch_on()
  enter(drive); IF loop not already sounding THEN play(snd_engine, looped)
  place(snd_engine)

FUNCTION stop()             # deliberate shutdown
  IF state == off THEN RETURN
  enter(stopping); stop loop after its current buffer; play(snd_engine_stop); place it

FUNCTION stall()            # engine died on its own
  IF state == off THEN RETURN
  enter(stalling); stop loop after its current buffer; play(snd_engine_stop); place it

FUNCTION transmission_switch()
  IF snd_transmission exists AND state != off THEN play it once at the emitter

FUNCTION enter(s)
  state = s; time_state_start = now
```

```text
FUNCTION update()                                # once per frame while switched on
  IF state == off THEN RETURN
  CASE starting:
      place(snd_engine_start)
      IF loop already sounding
          update_drive()                         # pitch it
      ELSE IF time_state_start + engine_start_delay < now
          play(snd_engine, looped); update_drive()
      IF start transient has finished
          drive()                                # settle into the running state
  CASE drive:     update_drive()
  CASE stalling:  place(snd_engine_stop); IF it finished THEN switch_off()
  CASE stopping:  same as stalling
```

**Invariants** — `stopping` and `stalling` behave identically; they are distinguished only
so that the rest of the car can ask which one happened. The loop is always stopped
*deferred* (let the current buffer finish) rather than cut, because a hard cut on a looping
engine sample is audible as a click.

**Notes** — `switch_off` is the only path out of the automaton, and it always runs when the
stop transient ends, which guarantees the scheduler registration is released exactly once.

## engine pitch

**Contract** — the loop's playback rate tracks revolutions, clamped to a narrow band.

```text
FUNCTION update_drive()
  scale = 0.5 + 0.5 * current_rpm / rpm_at_peak_torque
  clamp scale to [0.5, 1.25]
  set playback rate of snd_engine to scale
  place(snd_engine)
```

**Notes** — the sample is authored at the peak-torque revolutions, which is why that is the
divisor: at peak torque the rate is exactly 1 and the sample plays as recorded. The band is
deliberately asymmetric — half speed down, a quarter up — because pitching a recorded engine
*up* exposes the resampling artefacts much faster than pitching it down.

## teardown

**Contract** — `destroy` switches off (releasing the scheduler registration) and then
releases all four sound handles. The order matters: releasing a handle that is still playing
and still registered leaves the mixer holding a dangling emitter.
