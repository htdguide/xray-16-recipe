# src/xrSound/SoundRender_Emitter.h

> Declares one playing instance of a sound: its state machine, its volume components, its streaming
> ring and its cursor.

**Needs** — [`SoundRender.h`](SoundRender.h.md) · [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [`SoundRender_Scene.h`](SoundRender_Scene.h.md) · [`Sound.h`](Sound.h.md)
**Used by** — [`SoundRender_Core.cpp`](SoundRender_Core.cpp.md) · [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) · [`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) · [`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md) · [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) · [`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md) · [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md) · [`SoundRender_Target.cpp`](SoundRender_Target.cpp.md) · [`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md)
**Tier floor** — T1: it declares the raw PCM blocks handed to the device seam.

## Purpose

Declares the emitter, whose substance is split across three files:
[`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) (states, volume, ranking),
[`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md) (streaming, cursor, AI events) and
[`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md) (lifetime). It
implements the emitter interface the game sees, declared in [`Sound.h`](Sound.h.md).

## Exported units

The emitter's fields are public because the voice and the statistics pass read them directly; a
rebuild should narrow that, and the twins above say which reads are real.

- **`State`** — the nine-state enumeration; see the FSM twin for the graph.
- **`voice`, `scene`, `owner`** — the emitter's three links: the hardware voice it holds (or none),
  the world it lives in, the handle the game holds.
- **`importance`** — the caller's ranking scale, 100 for 2D sounds.
- **`smooth_volume`, `occluder_volume`, `fade_volume`** — the three independently smoothed volume
  components. Their rates and purposes are tabulated in the FSM twin.
- **`occluder`** — the three vertices of the last triangle found between listener and source,
  cached so the next frame's occlusion test can try one triangle before querying the database.
- **`params`** — position, base volume, instance volume, frequency, min/max distance, AI distance:
  the record the game reads back and the voice pushes at the device.
- **`time_started`, `time_to_stop`, `time_to_announce`, `seek_to_seconds`** — the emitter's clock.
  `time_to_stop` carries an infinite sentinel for a looped sound, which is also how the seek path
  recognises a loop.
- **`epoch`** — the update marker that guarantees one advance per frame.
- **`paused_at`** — the pause depth that silenced this emitter, or zero.
- **`ignores_time_factor`** — subscribes the emitter to the unscaled clock.
- **`is_2D`, `moved`, `stopping`, `rewind_requested`** — the state flags the FSM branches on.
- **`obtain_block`, `fill_*`, `dispatch_prefill`, `discard_prefilled_blocks`, `wait_prefill`** — the
  streaming ring; contracts in [`SoundRender_Emitter.cpp`](SoundRender_Emitter.cpp.md).
- **`cursor` accessors** — absolute (concatenated) and relative (within the current asset) forms.

## Notes

The infinite stop-time sentinel is the largest representable value of a 32-bit unsigned integer
converted to a real, and the seek path recognises a loop by comparing against that exact value.
That is a representation accident of the original; a rebuild should carry an explicit optional or
a loop flag and will then not need the sentinel at all.
