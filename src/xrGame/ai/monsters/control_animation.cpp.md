# src/xrGame/ai/monsters/control_animation.cpp

> Reconciles "the clip this creature should be playing" with "the blend actually running on its skeleton", once a frame, for three body slices — and raises the signals that let a blow land on the right frame.

**Needs** — [`control_animation.h`](control_animation.h.md) · [`base_monster.h`](basemonster/base_monster.h.md) · [`control_manager.h`](control_manager.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`control_animation.h`](control_animation.h.md)
**Tier floor** — T1: holds handles into the renderer's live blend list and writes their playback speed and position in place

## Purpose

Every decision a creature makes eventually becomes a clip, and this is where a clip becomes motion. The component is deliberately dumb: it does not choose clips, it does not know what a creature is, and it has no opinion about behaviour. Its whole responsibility is a reconciliation loop and a signal mechanism.

The split from [`control_animation_base.cpp`](control_animation_base.cpp.md) is worth stating because it is not obvious: *this* component owns the renderer contact and nothing else; the base owns the creature's animation *table* — which clip stands for which logical motion, which action plays which motion, which transitions exist. A rebuild that merges them ends up with the table depending on the renderer.

## State

```text
RECORD ControlAnimation
  skeleton            : handle to the creature's animated visual
  events              : map<clip, list<AnimationEvent>>
  frozen              : bool
  saved_speeds        : three reals, one per body slice
  ended_flags         : three bools, set from playback-end callbacks

RECORD AnimationEvent
  time_fraction : real   # 0..1 of the clip's duration
  signal_id     : int
  handled       : bool   # reset every time the clip is restarted
```

**Invariants** — `handled` is per (clip, event) and is cleared whenever that clip is started, which is what makes an event fire once per playthrough rather than once ever. The saved speeds are only meaningful while frozen or while a speed override is in force.

## `update_frame`

**Contract** — One frame of work, skipped entirely while frozen. Advances the renderer's animation tracks, drains the playback-end flags into notifications, starts any clip whose request does not match what is playing, then tests each slice's events. Runs per creature per frame; the profiler markers around its four phases are there because this is a hot path.

```text
FUNCTION update_frame()
  IF frozen   RETURN
  skeleton.advance_tracks()
  drain_end_flags()
  start_pending_clips()
  check_events(whole_body) ; check_events(torso) ; check_events(legs)
```

**Notes** — Advancing the tracks first means the events tested at the end of the frame reflect this frame's playback position, not last frame's. The source notes that the track advance belongs in a coarser update than the frame; it has not moved.

## `start_pending_clips`

**Contract** — Starts each slice whose requested clip differs from the running one, then applies the speed override to the whole-body slice only.

```text
FUNCTION start_pending_clips()
  FOR EACH slice IN (whole_body, legs, torso)
     IF NOT slice.actual
        start(slice)
  IF whole_body has a blend
     whole_body.blend.speed = (requested_speed > 0) ? requested_speed : saved_speed
```

**Notes** — Only the whole-body slice takes a speed override. That is a real constraint on the rest of the creature code: a behaviour can make a creature move faster, but it cannot make its legs and torso run at different speeds.

## `start(slice)` — the interesting part

**Contract** — Starts one clip on one slice, preserving phase across the change. Resolves which bone group the clip belongs to, falls back to the default group when the clip does not name one, captures the outgoing blend's phase, starts the new blend with a completion callback, restores the phase, stamps the start time, notifies the control manager, informs the footstep system, and re-arms the clip's events.

```text
FUNCTION start(slice, end_callback)
  group = clip.declared_bone_group
  IF group is unset   group = default_group

  phase = -1
  IF slice has a blend AND that blend loops
     phase = (blend.position modulo blend.duration) / blend.duration

  slice.blend = skeleton.play_cycle(group, clip, mixing = true,
                                    callback = end_callback, param = self)

  IF phase > 0 AND the new blend loops
     slice.blend.position = slice.blend.duration * phase

  slice.started_at = now()
  slice.actual     = true
  notify(animation_started)
  IF clip != torso's clip
     footstep_system.on_animation_start(clip, slice.blend)
  re-arm every event registered on `clip`
```

**Notes** — Phase preservation is the decision that makes creature movement look continuous. Switching from a walk to a run mid-stride would otherwise restart the new cycle from its first frame and the creature would visibly stutter; carrying the *fraction* across means the legs keep their place in the gait. It is applied only between looping clips, because a one-shot clip started halfway through would skip its beginning.

The footstep notification is suppressed for the torso slice, because footsteps come from the legs and the whole body, and notifying twice would double the sounds.

## `add_anim_event`

**Contract** — Registers a signal at a fraction of a clip's duration. Silently ignores a registration whose time is already occupied on that clip, so a clip cannot accumulate duplicate signals from repeated loads.

## `check_events`

**Contract** — For the slice's running clip, fires every registered, unfired event whose time fraction has been passed. Computes the elapsed fraction from wall-clock time since the clip started, against the clip's duration *divided by its playback speed*.

```text
FUNCTION check_events(slice)
  IF slice has no valid clip or no blend or is not actual   RETURN
  elapsed_fraction = (now() - slice.started_at)
                     / ((blend.duration / blend.speed) * 1000)
  FOR EACH event ON slice.clip
     IF NOT event.handled AND event.time_fraction < elapsed_fraction
        event.handled = true
        notify(animation_signal, { clip, event.time_fraction, event.signal_id })
```

**Notes** — Measuring elapsed time against the wall clock rather than reading the blend's own position is what makes an event fire *once* even if the clip loops several times without being restarted — the fraction keeps growing past one. Dividing the duration by the speed is what keeps a sped-up clip's events landing at the right moment; a rebuild that forgets it makes fast attacks miss their hit frame.

The event is fired *after* the moment it names, on the first frame that has passed it, never before. At thirty frames a second that is up to a frame of lateness, which the melee system absorbs by testing distance at the moment the signal arrives rather than trusting the animation.

## `freeze` / `unfreeze`

**Contract** — Save each slice's playback speed and set it to zero; restore the saved speeds. Idempotent in both directions. While frozen the whole frame update is skipped, so events do not fire and the tracks do not advance.

**Notes** — Zeroing the speed rather than detaching the blends means the pose holds exactly where it was and resumes without a restart. This is what a creature caught mid-animation by a script or a cut-scene does.

## `restart`

**Contract** — Re-resolves the skeleton handle and re-issues every active slice's clip against it, restoring each blend's absolute playback position. Used when the creature's visual is replaced underneath it.

**Notes** — Position is restored as an absolute time here, not as a fraction as in `start` — the clip is the same clip, so its duration has not changed, and preserving the exact time avoids any drift.

## `motion_time`

**Contract** — A free query, independent of any creature: how long a clip runs, in seconds, at its authored speed. Given a clip identity and a visual, returns the clip's length divided by its authored speed multiplier.

**Notes** — This is the value every timed behaviour state in the chapter builds its deadlines from. The authored speed multiplier belongs to the clip's definition in the model data, so changing it in the data changes every behaviour that waits for that clip — the timing is authored, not coded.

## `reinit` / `reset_data`

**Contract** — `reinit` re-resolves the skeleton, drops every registered event, clears the end flags and unfreezes. `reset_data` clears the three slices and sets the speed override to negative, meaning "use each clip's own speed".

**Notes** — Dropping the events on re-initialisation matters because they are registered from the creature's configuration at load; a creature re-initialised without clearing them would accumulate a duplicate set, which the duplicate guard in `add_anim_event` then silently absorbs.
