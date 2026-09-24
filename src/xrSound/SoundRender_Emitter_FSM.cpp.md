# src/xrSound/SoundRender_Emitter_FSM.cpp

> The emitter's state machine, the volume model, and the ranking function that decides who is
> audible — the core of the chapter.

**Needs** — [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [`SoundRender_Target.h`](SoundRender_Target.h.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md) · [`xrServerEntities/ai_sounds.h`](../xrServerEntities/ai_sounds.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: real-valued per-frame work over a small record; it runs inside the frame
budget but touches no device and no byte layout.

## Purpose

An emitter is one playing instance of a sound. It exists from the moment the game asks for a sound
until that sound's natural end or an explicit stop — *whether or not it is ever audible*. Whether
it is audible is decided here, every update, against the thirty-two voices the device gave us.

The central idea: **playing and being heard are different states.** An emitter that loses its voice
does not stop; it *simulates*, advancing its clock and its stream cursor in silence, and can be
promoted back mid-sound without a restart. Everything else in this file exists to make the
promotion and demotion inaudible.

## The state machine

```text
ENUM EmitterState
  Stopped
  StartingDelayed          # a play request with a delay, one-shot
  StartingLoopedDelayed    # ... looped
  Starting                 # first update after the delay elapsed, one-shot
  StartingLooped
  Playing                  # holds a voice, submitting audio, one-shot
  PlayingLooped
  Simulating               # no voice, clock and cursor still advancing, one-shot
  SimulatingLooped
```

Three facts are encoded twice — once in the state and once in a flag — and both encodings matter:

- **Looped or not** is baked into the state name because it changes what "the end of the data"
  means to the streaming fill (wrap, versus zero-pad and stop).
- **Delayed** is a separate entry state because a delayed sound must not reserve anything, claim a
  voice, or raise an AI hearing event while it waits.
- **Stopped** is terminal. The update pass in
  [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) destroys any emitter that
  reports it.

```text
                 play(delay>0)                 delay elapses
  Stopped ────────────────────► StartingDelayed ─────────────► Starting
     ▲            play(delay=0)                                   │
     │ ─────────────────────────────────────────────────────────► │
     │                                        rank wins a voice   │  rank loses
     │                            ┌──────────────────────────────┴──────────┐
     │                            ▼                                         ▼
     │                        Playing ◄──── rank wins again ──────── Simulating
     │                            │  ──── rank lost / paused ───────►    │
     └──── one-shot: clock passed its stop time; or explicit stop ────────┘
```

Looped emitters run the same graph with the looped states, minus the "clock passed its stop time"
edge — a loop's stop time is an infinite sentinel and only an explicit stop ends it.

## `update`

**Contract** — Advances the emitter by one frame against the clock it subscribes to (scaled or
steady). Runs exactly once per emitter per update, driven by the processor. May take a voice, may
lose one, may raise an AI hearing event, may block briefly on the streaming worker when it has to
discard or rebuild the prefilled blocks. Leaves the emitter in `Stopped` if it finished, which is
the signal for the processor to destroy it.

The pass is four things in order, and the order is load-bearing.

```text
FUNCTION update(time, dt)
  # 1. A pending seek, applied before any state work.
  IF rewind_requested THEN
    wait for the streaming worker
    shift time_started / time_to_stop / time_to_announce by the elapsed gap
    cursor ← 0
    IF holding a voice THEN refill every block, rewind the voice, dispatch a refill
    rewind_requested ← false

  # 2. The state machine (below).
  step_state(time, dt)

  # 3. A pending absolute seek to a requested second, applied in any live state.
  IF seek_to_seconds > 0 THEN apply_seek(time)

  # 4. A deferred stop completes once the fade has actually reached silence.
  IF stopping AND fade_volume IS zero THEN hard_stop()

  # Footer.
  moved ← false
  IF state ≠ Stopped THEN
    IF time ≥ time_to_announce THEN announce_to_ai()
  ELSE
    release the owning handle's back-pointer and drop the handle
```

Step 4 is the whole meaning of a *deferred* stop: the caller says "stop", the emitter keeps
rendering while its fade ramp walks to zero over about a tenth of a second, and only then does the
voice come back. A non-deferred stop skips the ramp and clicks.

### The `Starting` transition

```text
ON Starting (or StartingLooped)
  IF paused THEN stay
  time_started      ← time
  time_to_stop      ← IF looped THEN INFINITE ELSE time + length_sec / frequency
  time_to_announce  ← time
  fade_volume       ← 1
  occluder_volume   ← scene.occlusion_at(position, 0.2)   # one ray, see Scene
  smooth_volume     ← base × volume × slider × (IF 2D THEN 1 ELSE occluder_volume)
  environment       ← scene.environment_at(position)      # both current and target
  IF rank_wins_a_voice(dt) THEN
    state ← Playing; cursor ← 0; take_voice(self); dispatch refill
  ELSE
    state ← Simulating
```

Two decisions here. The stop time is divided by the playback frequency, so a sound played at double
pitch ends in half the time — pitch is a playback *rate*, and the emitter's own clock must agree
with the device's or a one-shot would be cut off or padded. And the smoothed volume is *seeded*
rather than filtered on the first frame: without the seed the one-pole filter below would start at
its initial value and take a tenth of a second to reach the real volume, so every sound would fade
in.

### `Playing` → `Simulating` and back

A playing emitter loses its voice when its rank falls below every voice's, when it is paused, or
when it is culled for being too quiet. It stops for good only when its clock passes its stop time.
On demotion the voice is released *and the prefilled blocks are discarded* — they hold audio for a
cursor position that will be stale by the time the emitter is promoted again.

A simulating emitter does one extra thing a playing one does not: it recomputes its stream cursor
from wall-clock time.

```text
FUNCTION cursor_from_clock(time_started, time, length_sec, frequency, info) -> int
  IF time < time_started THEN time ← time_started    # a pause can leave the clock behind
  WHILE (time - time_started) > length_sec / frequency
    time ← time - length_sec / frequency             # fold a looped sound into one period
  sample ← floor((time - time_started) × frequency × info.samples_per_sec)
  RETURN sample × info.bytes_per_sample × info.channels
```

This is the promotion machinery: a sound that spent four seconds inaudible resumes four seconds in,
and a looped ambient bed resumes at the right point in its loop. A rebuild that tracks the cursor by
accumulating decoded bytes instead will drift whenever an emitter is demoted, because no bytes are
decoded while it is silent.

## `update_culling` — the volume model

**Contract** — Recomputes the emitter's volume from distance, occlusion and fading, and returns
whether it should hold a voice this frame. Side-effect-heavy by design: it is the only place the
three volume components move. May cast one occlusion ray. Returns false both for "too far" and for
"too quiet to be worth a voice", and the caller treats both as demotion.

```text
FUNCTION update_culling(dt) -> bool
  IF is_2D THEN
    occluder_volume ← 1                          # a UI sound is never occluded
    fade_volume ← fade_volume + dt × 10 × (IF stopping THEN -1 ELSE +1)
  ELSE
    distance ← |listener.position - position|
    IF distance > max_distance THEN
      smooth_volume ← 0
      RETURN false                               # hard cutoff, no ray, no smoothing

    attenuation ← clamp(min_distance / (rolloff × distance), 0, 1)
    instant ← attenuation × base × volume × slider
    # Fade *out* while stopping or while too quiet to matter; otherwise fade in.
    fade_volume ← fade_volume + dt × 10 ×
                  (IF stopping OR instant < cull_volume THEN -1 ELSE +1)

    # World ambience is exempt from occlusion: it has no position to be behind anything.
    occ ← IF game_type IS WORLD_AMBIENT THEN 1 ELSE scene.occlusion_at(position, 0.2)
    occluder_volume ← approach(occluder_volume, occ, rate 1.0, dt)
    clamp occluder_volume to [0,1]

  clamp fade_volume to [0,1]
  smooth_volume ← 0.9 × smooth_volume
                + 0.1 × (base × volume × slider × occluder_volume × fade_volume)

  IF smooth_volume < cull_volume THEN RETURN false    # but keep simulating: it may rise again

  IF holding a voice THEN
    voice.rank ← rank()                               # refresh so challengers see the truth
    RETURN true
  RETURN may_play(self)
```

There are four separate smoothing mechanisms stacked here and each has a different job:

| mechanism | rate | what it prevents |
|---|---|---|
| `fade_volume` ramp | 0 → 1 in 0.1 s | clicks when a voice starts, stops or is stolen |
| `occluder_volume` approach | 1.0 per second | a doorway's occlusion snapping as the listener steps through |
| `smooth_volume` one-pole | ≈0.1 per frame | rank chatter at the cull threshold |
| strict `<` in `may_play` | — | two equal-ranked emitters trading a voice every frame |

`slider` is the player's effects volume times the engine's effects trim for an effect sound, or the
player's music volume for a music sound — which is the entire reason a sound carries a type.

The `distance > max_distance` early exit is what makes this affordable: the expensive part of the
function is the occlusion ray, and nothing beyond the source's own authored range casts one.

## `rank`

**Contract** — The single number every voice decision compares. Pure; no side effects.

```text
FUNCTION rank() -> real
  distance ← |listener.position - position|
  attenuation ← clamp(min_distance / (rolloff × distance), 0, 1)
  RETURN smooth_volume × attenuation × importance
```

Distance enters twice — once inside `smooth_volume` and again as a bare factor — which sharpens the
near/far ordering beyond what loudness alone gives. Two sounds mixed to the same apparent loudness,
one close and quiet and one distant and loud, are not equally important: the close one is the one
the player is standing in, and this is how the ranking says so.

`importance` is the per-emitter scale the game sets. Two-dimensional sounds set it to 100 when they
switch to 2D, which puts them above every positional sound in the world — a rebuild could express
that as a priority class instead, but the effect must be the same: the user interface and the
player's own weapon are never culled.

## `update_environment`

**Contract** — Keeps a per-emitter reverb state blending toward the preset of the region the
*source* sits in. The region is re-queried only when the emitter moved, since a stationary source
cannot change rooms. Uses the same frame-delta-as-blend-factor as the listener's blend.

**Notes** — Per-emitter reverb is computed but, in the shipped OpenAL backend, not applied: the
reverb seam the engine actually uses is listener-global, one preset for the whole mix. A rebuild
targeting a mixer with per-source reverb sends has this value ready. Until then it is dead
computation and may be dropped.

## `render`

**Contract** — Called once per frame for each emitter holding a voice, from the render pass. Pushes
the 3D parameters at the voice, then either starts it (first frame) or services its buffer queue,
then asks the streaming worker to top the prefill back up.

```text
FUNCTION render()
  voice.push_parameters()
  IF voice.is_streaming THEN voice.service_queue() ELSE voice.begin_streaming()
  dispatch_prefill()
```

The first-frame split is what gives the decoder a frame of head start: a voice is claimed in the
update pass and only begins submitting in the *next* render, by which time the worker has filled
its blocks.
