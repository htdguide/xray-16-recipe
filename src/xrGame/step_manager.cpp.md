# src/xrGame/step_manager.cpp

> Turns a walk animation into footsteps: sounds, dust and a camera shake, fired at authored
> times within the animation rather than driven by the feet.

**Needs** — [`step_manager.h`](step_manager.h.md) · [`step_manager_defs.h`](step_manager_defs.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`material_manager.h`](material_manager.h.md) · [`IKLimbsController.h`](IKLimbsController.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`Level.h`](Level.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`step_manager.h`](step_manager.h.md)
**Tier floor** — T2: a per-frame time comparison against an authored schedule, plus the
audio and particle handoffs.

## Purpose

A creature's feet touch the ground at particular moments inside a walk cycle. Those moments
are **authored per animation, per leg**, in configuration — not detected from the skeleton —
and this module fires the consequences when the clock reaches them.

That is the load-bearing decision of the file, and it is worth defending. The alternative —
watch each foot bone and fire when it crosses the ground plane — sounds more principled and
is worse: it costs a per-frame bone-transform read per leg, it fires on animation blending
artifacts, and it gives a designer no way to make a limp sound like a limp. Here the
designer writes the phase of each footfall as a fraction of the cycle, and the engine
believes them. Foot *bones* are still needed, but only to place a dust particle, not to
decide when.

A step produces three things: a sound chosen from the material pair under the creature, a
particle burst at the foot, and a game-layer event that a subclass uses for a camera shake.

## State

```text
RECORD StepManager
  legs_count        : int                  # 1..4; a biped is 2, a dog is 4
  steps             : map<Motion, StepParams>   # per animation, the authored schedule
  info              : StepInfo             # the currently playing animation's state
  foot_bones        : Bone[4]              # which skeleton bone is each leg's foot
  blend             : optional<Blend>      # the animation currently playing
  time_anim_started : int (ms)
  step_sound        : MaterialSoundState   # last sound index and last material pair
```

The two record shapes in [`step_manager_defs.h`](step_manager_defs.h.md) carry the
substance:

```text
RECORD StepParams                          # authored, per animation
  step   : { time : real, power : real }[4]   # per leg: phase within one cycle, and loudness
  cycles : int                             # how many footfall cycles this clip contains

RECORD StepInfo                            # live, for the animation now playing
  activity : { handled : bool, cycle : int }[4]   # per leg: has this cycle's step fired yet
  params   : StepParams
  disable  : bool                          # this animation has no authored schedule
  cur_cycle: int                           # 1-based
```

**Invariants**

- `step[i].time` is a fraction **of one cycle**, not of the clip. A clip declaring three
  cycles has its footfalls repeat three times across its length, so one authored schedule
  serves a clip of any length.
- Cycles are numbered from one. Zero would be indistinguishable from "not yet started" in
  the per-leg record.
- `disable` is the state for an animation with no entry in the table: the manager goes
  quiet rather than guessing. Most animations have no schedule.
- The number of foot bones found must equal `legs_count`, checked at load.

## `reload(section)`

**Contract** — loads the whole authored schedule for one creature class from configuration.
Blocks; runs once per creature at construction. Tolerates a missing schedule section (the
creature simply never makes footstep sounds) but not a malformed one.

```text
FUNCTION reload(section)
  legs_count := config.int(section, "LegsCount")            # 1..4
  schedule_section := config.text(section, "step_params")
  IF schedule_section DOES NOT EXIST
    RETURN                                                  # silent creature

  FOR EACH line (animation_name, values) IN schedule_section
    params.cycles := values[0]                              # must be at least 1
    FOR leg IN 0 .. legs_count - 1
      params.step[leg].time  := values[1 + leg * 2]
      params.step[leg].power := values[1 + leg * 2 + 1]
    motion := skeleton.find_cycle(animation_name)
    IF motion IS none
      CONTINUE                                              # animation not in this model
    steps[motion] := params

  foot_bones := all none
  reload_foot_bones()
  time_anim_started := 0
  blend := none
```

**Invariants** — the line format is positional: cycle count, then a time/power pair per leg,
in leg order. The number of pairs read is the creature's leg count, so the same
configuration file cannot be shared between a biped and a quadruped. This format is in the
shipped game data and is frozen.

Skipping an animation the model does not have is deliberate and important: one schedule
section is shared by several creature models, and each takes the subset it can play.

**Notes** — the schedule is keyed by the *resolved* animation identity rather than by name,
so the lookup at play time is a map probe rather than a string comparison. A rebuild must
resolve names at load for the same reason.

## `reload_foot_bones`

**Contract** — finds the four (or fewer) foot bones by name. The names come from the model's
own embedded configuration if it has a `foot_bones` section, and otherwise from a section
named by the creature's configuration. Fails hard if neither exists, and asserts that
exactly `legs_count` bones were found.

**Notes** — model-first is the right precedence: which bone is the front-left foot is a
property of the *mesh*, not of the creature class, and several creature classes share a
mesh. The fallback exists for models authored before the convention.

Failing hard here rather than degrading is a deliberate choice, and a reasonable one: a
creature with the wrong foot bones puts dust clouds in the air beside it, which is much
harder to notice in review than a hard failure at load.

## `on_animation_start(motion, blend)`

**Contract** — called when the animation layer starts a new clip. Rebinds the manager to the
new clip, resets the per-leg firing record, and hands the clip to the inverse-kinematics
limb controller if the creature has one.

```text
FUNCTION on_animation_start(motion, blend)
  self.blend := blend
  IF blend IS none
    RETURN
  ik_controller.play_legs(blend)              # the same clip drives foot placement
  time_anim_started := now

  IF steps HAS motion
    info.params    := steps[motion]
    info.disable   := false
    info.cur_cycle := 1
    FOR leg IN 0 .. legs_count - 1
      info.activity[leg] := { handled: false, cycle: 1 }
  ELSE
    info.disable := true                      # no schedule: stay silent
```

**Invariants** — the order matters: the start time is stamped **before** the schedule
lookup, so that a clip with a schedule and a clip without are timed identically. The per-leg
records are cleared to the first cycle, not to zero, matching the one-based numbering.

## `update(is_first_person)`

**Contract** — one frame. Fires every footfall whose authored time has passed and which has
not yet fired in the current cycle; advances the cycle counter; and restarts the whole
schedule when a looping clip wraps. Does nothing for a clip with no schedule or no active
blend.

```text
FUNCTION update(is_first_person)
  IF info.disable OR blend IS none
    RETURN

  audible := distance_sqr(my_position, camera_position) < 400      # 20 world units
  cycle_time := clip_duration / info.params.cycles                 # duration is clip length / playback speed
  material_pair := none;  material_resolved := false

  FOR leg IN 0 .. legs_count - 1
    IF info.activity[leg].handled AND info.activity[leg].cycle == info.cur_cycle
      CONTINUE                                                     # already fired this cycle
    due := time_anim_started
           + 1000 * (cycle_time * (info.cur_cycle - 1)
                     + cycle_time * info.params.step[leg].time)
    IF due > now
      CONTINUE

    IF NOT material_resolved
      material_pair := material_under_me()                         # resolved at most once per frame
      material_resolved := true
    IF material_pair IS none
      BREAK                                                        # no ground: no steps at all

    IF audible AND is_on_ground()
      play_step_sound(material_pair, info.params.step[leg].power, is_first_person)
    IF audible AND material_pair HAS collide_particles
      spawn_particle(random of material_pair.collide_particles, at foot_position(leg), facing up)
    on_step()                                                      # the subclass hook: camera shake

    info.activity[leg] := { handled: true, cycle: info.cur_cycle }

  IF info.cur_cycle < info.params.cycles
    info.cur_cycle := 1 + floor((now - time_anim_started) / (1000 * cycle_time))

  clip_end := time_anim_started + 1000 * clip_duration
  IF clip loops AND clip_end < now
    time_anim_started := clip_end                                  # not `now`: no drift
    info.cur_cycle := 1
    reset every leg's activity to cycle 1
```

**Invariants** — five things here are decisions.

*The due time is absolute, derived from the clip's start stamp and its real duration* — clip
length divided by playback speed, so a creature playing a walk at reduced speed steps more
slowly, automatically.

*A footfall fires at or after its time, never before, and at most once per leg per cycle.*
The per-leg record is keyed by cycle number, not a plain flag, which is what makes a
multi-cycle clip fire each leg once per cycle rather than once per clip.

*A footfall that is late still fires*, on the frame the manager notices. A frame hitch moves
a footstep sound a frame or two but never drops it.

*The audibility cut is at twenty world units and gates sound and particles but not the
game-layer event.* The creature is not simulated differently far away — it simply stops
producing effects nobody can perceive. The distance is compared squared to avoid a square
root per creature per frame; that is incidental, the twenty units is not.

*A looping clip restarts from its own computed end time, not from the current frame time.*
This is the one line that prevents drift: restarting from `now` would add a fraction of a
frame to every lap, and after a minute of walking the footsteps would be audibly out of step
with the animation.

The material under the creature is resolved lazily and at most once per frame, because it is
a collision query and most frames fire no footfall at all. A creature over no material —
falling, clipped out of the world — fires nothing.

## `get_foot_position(leg)`

**Contract** — the world position of one foot, as the creature's transform composed with
that bone's current pose. Fails if the bone was never found. Used only to place a particle.

**Notes** — this is the module's one read of the *animated* skeleton, and it happens only on
the frame a step fires, which is what keeps the schedule-driven design cheap.

## `play_next(material_pair, volume, is_first_person)`

**Contract** — plays one footstep sound from the set the material pair offers, choosing a
different one from last time. Silent if the pair has no sounds.

```text
FUNCTION play_next(pair, volume, first_person)
  IF pair.step_sounds IS empty
    RETURN
  position := my_position lifted half a unit          # ear height, not ankle height
  IF pair IS NOT last_pair OR nothing played yet
    index := random over all sounds                   # new surface: no history to avoid
    last_pair := pair
  ELSE
    index := (last_index + 1 + random over (count - 1)) MOD count   # any sound but the last
  IF first_person
    volume := volume * hud_step_volume_setting
    position := the listener's own position           # a 2D sound, not placed in the world
  play(pair.step_sounds[index], at position, volume)
```

**Invariants** — the selection rule is *uniform over everything except the previous sound*.
That is not the same as uniform random, and the difference is exactly the point: the human
ear detects an immediately repeated footstep sound instantly, and a plain random choice
repeats one time in N. The offset-then-wrap arithmetic is a constant-time way to draw from
"all but one".

The history is reset when the surface changes, because the first step on gravel after a step
on metal has no repetition to avoid.

First-person steps are rendered as non-positional sound at the listener and scaled by a
separate user volume setting, because the player's own footsteps must not pan or attenuate
with head movement.

**Notes** — the half-unit lift on the emitting position is the module's only spatial
approximation: footsteps are emitted from the creature's middle rather than from the foot
that made them. For a sound whose direction cue is dominated by the creature's own position,
the simplification is free.

## `on_step` (the subclass hook)

**Contract** — empty here; a creature class overrides it to shake the camera or to notify
whatever else cares. Called for every footfall, including inaudible ones, which is correct:
a camera shake is a first-person effect and the player's own creature is never distant.

## `is_on_ground` (the subclass hook)

**Contract** — true here; a creature class overrides it to suppress step sounds while
airborne. Gates the sound but not the particle, which is an asymmetry with no recoverable
reason.

## What could not be recovered

- The twenty-unit audibility cut and the half-unit sound-emitter lift are bare numbers with
  no derivation anywhere in the source.
- Why `is_on_ground` gates the footstep sound but not the dust particle.
