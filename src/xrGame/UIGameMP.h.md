# src/xrGame/UIGameMP.h

> Declares the layer every multiplayer mode's interface shares — the server greeting and the recorded-match controls — implemented in [`UIGameMP.cpp`](UIGameMP.cpp.md).

**Needs** — [`UIGameCustom.h`](UIGameCustom.h.md)
**Used by** — [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`UIGameCTA.h`](UIGameCTA.h.md) · [`UIGameDM.cpp`](UIGameDM.cpp.md) · [`UIGameDM.h`](UIGameDM.h.md) · [`UIGameMP.cpp`](UIGameMP.cpp.md) · [`game_cl_mp.cpp`](game_cl_mp.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the thin multiplayer layer between the in-game interface base and the individual
modes. Substance in [`UIGameMP.cpp`](UIGameMP.cpp.md).

Exported units:

- `UIGameMP` — the layer. Owns two windows and nothing else.
- `ShowServerInfo`, `IsServerInfoShown`, `SetServerLogo`, `SetServerRules` — the server's
  greeting screen: a logo and a rules text pushed from the server, which gates the start of
  the match. A server with nothing to say acknowledges on the client's behalf.
- `ShowDemoPlayControl` — the recorded-match playback controls, created on first use and
  reopened with the cursor where it was left.
- `IR_UIOnKeyboardPress`, `IR_UIOnKeyboardRelease` — intercepts exactly one action: the
  crouch action opens the playback controls while watching a recording.
- `SetClGame` — bind to a new session; rebuilds the greeting window, because its content
  belongs to the server just connected to.
