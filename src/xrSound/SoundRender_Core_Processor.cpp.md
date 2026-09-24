# src/xrSound/SoundRender_Core_Processor.cpp

> The per-frame passes: advance every emitter's clock and state, blend the listener's reverb, feed
> the voices, and publish the statistics the debug overlay reads.

**Needs** — [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`SoundRender_Emitter.h`](SoundRender_Emitter.h.md) · [`SoundRender_Target.h`](SoundRender_Target.h.md) · [`SoundRender_Source.h`](SoundRender_Source.h.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md) · [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md) · [Seam: Profiler and GPU debugging](../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a scheduling pass over object lists; nothing here is device- or layout-facing.

## Purpose

Sound runs in two passes per frame, and the separation is the point of this file. **Update** is
simulation: it advances clocks, runs each emitter's state machine, decides who deserves a voice,
and queues the AI hearing events. **Render** is submission: it hands the device fresh audio for the
voices that survived. Update may run when rendering is disabled (a dedicated server still needs AI
hearing events); render never runs without a preceding update.

## `update`

**Contract** — Advances the whole sound world by one frame. Takes the listener's transform. Sets
the re-entry lock for its duration, so any attempt to start or stop a sound from inside an emitter
callback is caught rather than corrupting the lists. Does not block on the streaming worker except
where an emitter's own state change demands it.

```text
FUNCTION update(position, forward, up, right)
  IF NOT ready THEN RETURN
  locked ← true

  clock.time_factor ← time_factor           # game speed affects pitch and duration
  clock_delta ← advance(clock)
  clock_steady_delta ← advance(clock_steady)

  emitter_epoch ← emitter_epoch + 1

  # Pass 1: everyone who currently holds a voice, in voice order.
  FOR EACH voice IN targets
    IF voice.emitter EXISTS THEN advance_emitter(voice.emitter)

  # Pass 2: everyone else, per scene; reap the dead as we go.
  FOR EACH scene IN scenes
    FOR EACH emitter IN scene.emitters          # index walk: the list shrinks
      IF emitter.epoch ≠ emitter_epoch THEN advance_emitter(emitter)
      IF NOT emitter.is_playing THEN
        destroy emitter
        remove it from scene.emitters

  update_listener(position, forward, up, right, clock_delta)

  FOR EACH scene IN scenes
    scene.dispatch_events()                     # AI hearing, see Scene

  locked ← false

FUNCTION advance_emitter(e)
  # An emitter may opt out of game-speed scaling: menu clicks must not
  # slow down when the world does.
  time  ← IF e.ignores_time_factor THEN clock_steady_value ELSE clock_value
  delta ← IF e.ignores_time_factor THEN clock_steady_delta ELSE clock_delta
  e.update(time, delta)
  e.epoch ← emitter_epoch
```

**Invariants** — Every emitter is advanced exactly once per update. That is what the epoch counter
buys: the voice-holders are updated *first*, deliberately, so that an emitter which is about to
lose its voice has already published its current rank before a challenger asks whether a voice is
free. Walking the scene list alone would advance them in creation order and let a newly created
emitter steal a voice from a holder whose rank had not yet been recomputed this frame.

An emitter that reports itself stopped is destroyed inside this walk, which is the only place
emitters die. That is why the walk is an index walk rather than an iterator walk — the list is
mutated during it.

## `render`

**Contract** — For each voice holding an emitter, let that emitter push parameters and refill the
device's buffer queue. Holds the re-entry lock. Runs after update in the same frame, and is skipped
entirely on a headless server.

```text
FUNCTION render()
  locked ← true
  FOR EACH voice IN targets
    IF voice.emitter EXISTS THEN voice.emitter.render()
  locked ← false
```

## `statistic`

**Contract** — Fills two optional reports. The compact one counts voices actually submitting audio,
emitters simulated across all scenes, and AI events raised last frame. The extended one describes
every emitter — name, parameters, smoothed volume, whether it is 3D, whether it holds a voice, and
its owning game object — and is what the in-game sound debugger draws. Allocates into the
caller's container; not thread-safe against update.

**Notes** — "Rendered" counts only voices that are both occupied *and* actually streaming, which
differs from "occupied": a voice is claimed one frame and begins submitting the next.

## `dump_statistics`

**Contract** — Writes the update and render timings plus the three counters into the debug overlay,
then rolls the timing accumulators over to the next frame. Purely diagnostic; a rebuild may omit it
with no behavioural consequence.
