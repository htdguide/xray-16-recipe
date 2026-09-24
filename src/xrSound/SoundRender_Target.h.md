# src/xrSound/SoundRender_Target.h

> Declares a hardware voice and the six operations a device backend must fill in.

**Needs** — [`SoundRender.h`](SoundRender.h.md)
**Used by** — [`SoundRender_Core_Processor.cpp`](SoundRender_Core_Processor.cpp.md) · [`SoundRender_Core_StartStop.cpp`](SoundRender_Core_StartStop.cpp.md) · [`SoundRender_Emitter_FSM.cpp`](SoundRender_Emitter_FSM.cpp.md) · [`SoundRender_Emitter_StartStop.cpp`](SoundRender_Emitter_StartStop.cpp.md) · [`SoundRender_Target.cpp`](SoundRender_Target.cpp.md) · [`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md) · [`SoundRender_TargetA.h`](SoundRender_TargetA.h.md)
**Tier floor** — T2: an abstract voice surface.

## Purpose

Declares the voice, whose device-independent half is in
[`SoundRender_Target.cpp`](SoundRender_Target.cpp.md) and whose filling is
[`SoundRender_TargetA.cpp`](SoundRender_TargetA.cpp.md). Together with the four abstract
operations on the manager, this class *is* the audio device seam — a rebuild targeting a different
mixer replaces these two files and nothing else.

## Exported units

- **`Voice`** — the base class.
- **`initialize` / `destroy` / `restart`** *(abstract)* — acquire, release and re-acquire the
  device resources backing one voice. `restart` exists for a device-lost path.
- **`start`** — bind an emitter; deferred, no audio yet.
- **`render`** — first submission; fill the queue and begin.
- **`update`** — recycle and refill consumed buffers.
- **`rewind`** — restart at the emitter's new cursor.
- **`stop`** — release the voice and mark it free.
- **`fill_parameters`** — push position, distances, gain and pitch.
- **`get_priority` / `set_priority`** — the rank the allocator compares, refreshed by the emitter
  every update it holds the voice.
- **`get_emitter` / `get_Rendering`** — the two read-only views the processor and the statistics
  pass need.
