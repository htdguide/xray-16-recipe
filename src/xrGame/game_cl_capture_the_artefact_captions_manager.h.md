# src/xrGame/game_cl_capture_the_artefact_captions_manager.h

> Declares the capture-the-artefact caption manager, implemented in [`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md).

**Needs** — [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`UIGameCTA.h`](UIGameCTA.h.md)
**Used by** — [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the object that owns every on-screen prompt in capture the artefact. Substance is in
[`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md).

The declaration's own load-bearing content is the **direction of the interface**: the setters
take *facts* (buying is possible, the paid spawn is affordable, this team won) and the single
show method derives every caption from them. There is no "hide this prompt" call, because
every prompt is cleared and re-derived each update.

Exported units:

- `CTAGameClCaptionsManager` — the manager. Holds the pushed facts, the two countdown
  strings, and the last whole second announced.
- `Init` — bind to the mode and its screen.
- `ShowCaptions` — the once-per-update pass: clear everything, then dispatch on phase.
- `ResetCaptions` — clear every caption.
- `CanCallBuy` / `CanCallBuySpawn` / `CanSpawn` — the pushed facts. The third is stored and
  never read.
- `SetWinnerTeam` — the end-of-round winner; the spectator value means "none yet".
- `SetWarmupTime` — rebuild the warm-up caption and report which second, if any, should be
  spoken aloud this update.
- `SetTimeLimit` — rebuild the remaining-time caption.
- `ShowInProgressCaptions` / `ShowPendingCaptions` / `ShowScoreCaptions` — the per-phase
  derivations. The pending one deliberately shows nothing.
- `ConvertTime2String` — milliseconds to a zero-padded clock.
