# src/xrSound/xr_streamsnd.h

> Declares the dead pre-Vorbis music streamer. Not compiled.

**Needs** — [`Sound.h`](Sound.h.md)
**Used by** — [`MusicStream.cpp`](MusicStream.cpp.md) · [`MusicStream.h`](MusicStream.h.md) · [`xr_streamsnd.cpp`](xr_streamsnd.cpp.md)
**Tier floor** — T1 as written: it names device buffer and codec handle types.

## Purpose

Declares the streamer described in [`xr_streamsnd.cpp`](xr_streamsnd.cpp.md), which is excluded
from the build. A rebuild implements neither file: music is an ordinary looped 2D emitter whose
sound type selects the music volume slider.

## Exported units

- **`SoundStream`** — one streamed music track: its file, its three volume levels (requested,
  smoothed, authored base), its loop count, its device buffer, its codec conversion stream, its
  write cursor and its decode position.
- **`play` / `stop` / `pause` / `is_playing` / `set_volume` / `restore` / `update` / `on_move`` —
  the control surface, mirroring what an emitter does today.
- Two RIFF container record shapes, read as memory images.

## Notes

The only idea here without an equivalent in the live code is the smoothed volume applied as
`half old plus half new` per update — a far coarser filter than the emitter's, and applied to a
logarithmic device gain rather than a linear one.
