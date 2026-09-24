# src/xrSound/xr_cda.h

> Declares the dead CD-audio player. Not compiled.

**Needs** — _(none)_
**Used by** — [`xr_cda.cpp`](xr_cda.cpp.md)
**Tier floor** — T4.

## Purpose

Declares the player described in [`xr_cda.cpp`](xr_cda.cpp.md). A rebuild implements neither file.

## Exported units

- **`CdPlayer`** — the reply buffer, the current track, the countdown watchdog and the working and
  paused flags.
- **`open` / `close` / `set_track` / `play` / `stop` / `pause` / `on_move`** — the control surface
  and the per-frame watchdog tick.
- **`CdState`** — playing, stopped, paused, tray open, not ready. Any unrecognised or failed reply
  reads as not ready, which is the safe default: the player then does nothing.
