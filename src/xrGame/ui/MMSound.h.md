# src/xrGame/ui/MMSound.h

> Declares the main menu's two sound channels: the spinner and the background music.

**Needs** — [`MMSound.cpp`](MMSound.cpp.md)
**Used by** — [`MMSound.cpp`](MMSound.cpp.md) · [`UIMMShniaga.cpp`](UIMMShniaga.cpp.md) · [`UIMMShniaga.h`](UIMMShniaga.h.md)
**Tier floor** — T2: owns audio handles with a defined release point

## Purpose

Declares the surface implemented in [`MMSound.cpp`](MMSound.cpp.md).

## `CMMSound`

The menu's audio. Configured from the menu's own layout document, which names the sounds
rather than hard-coding them.

- `Init(document, path)` — read the play list and the two spinner sounds.
- `whell_Play` / `whell_Stop` / `whell_Click` / `whell_UpdateMoving(frequency)` — the looping
  sound of the rotating menu spinner, its click, and the pitch that tracks its speed.
- `music_Play` / `music_Stop` / `music_Update` — the background track.
- `all_Stop` — release everything; also what destruction does.
