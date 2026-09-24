# src/xrGame/ui/UILoadingScreen.h

> Declares the game's loading screen and the do-nothing implementation the dedicated server uses.

**Needs** — [`UILoadingScreen.cpp`](UILoadingScreen.cpp.md) · [`xrEngine/ILoadingScreen.h`](../../xrEngine/ILoadingScreen.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`GamePersistent.cpp`](../GamePersistent.cpp.md) · [`UILoadingScreen.cpp`](UILoadingScreen.cpp.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`UILoadingScreen.cpp`](UILoadingScreen.cpp.md). Both
classes implement one engine-owned interface; the interface's own contract is at
[`ILoadingScreen.h`](../../xrEngine/ILoadingScreen.h.md).

## Exported units

- **`UILoadingScreen`** — the real one. It is simultaneously a widget and the engine's
  loading-screen service, which is the whole point: the engine drives it without knowing it
  is a widget tree.
  - `Initialize` — build from data, or from the compiled-in fallback.
  - `Show` / `IsShown` — hiding also releases the level logo's texture and clears the text.
  - `Update(completed, total)` — drive the bar and the percentage.
  - `Draw` — under the lock.
  - `SetLevelLogo`, `SetStageTitle`, `SetStageTip` — the three content hooks the loader calls.

- **`NullLoadingScreen`** — every method empty; always reports itself hidden. The dedicated
  server's implementation.
