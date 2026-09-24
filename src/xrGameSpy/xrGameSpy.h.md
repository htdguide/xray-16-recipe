# src/xrGameSpy/xrGameSpy.h

> The module's public face: the three identity queries, plus a single include point that
> pulls in every wrapper class so a consumer needs one import.

**Needs** — [`xrGameSpy.cpp`](xrGameSpy.cpp.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [`GameSpy_Available.h`](GameSpy_Available.h.md) · [`GameSpy_Browser.h`](GameSpy_Browser.h.md) · [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md) · [`GameSpy_QR2.h`](GameSpy_QR2.h.md) · [`GameSpy_GCD_Client.h`](GameSpy_GCD_Client.h.md) · [`GameSpy_GCD_Server.h`](GameSpy_GCD_Server.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`login_manager.h`](../xrGame/login_manager.h.md) · [`xrGameSpyServer.cpp`](../xrGame/xrGameSpyServer.cpp.md) · [`GameSpy_ATLAS.h`](GameSpy_ATLAS.h.md) · [`GameSpy_Available.h`](GameSpy_Available.h.md) · [`GameSpy_Browser.h`](GameSpy_Browser.h.md) · [`GameSpy_BrowsersWrapper.h`](GameSpy_BrowsersWrapper.h.md) · [`GameSpy_Full.h`](GameSpy_Full.h.md) · [`GameSpy_GCD_Client.h`](GameSpy_GCD_Client.h.md) · [`GameSpy_GCD_Server.h`](GameSpy_GCD_Server.h.md) · [`GameSpy_GP.h`](GameSpy_GP.h.md) · [`GameSpy_HTTP.h`](GameSpy_HTTP.h.md) · [`GameSpy_QR2.h`](GameSpy_QR2.h.md) · [`xrGameSpy.cpp`](xrGameSpy.cpp.md)
**Tier floor** — T3: a declaration surface.

## Purpose

Declares the identity functions implemented in [`xrGameSpy.cpp`](xrGameSpy.cpp.md) and
re-exports the wrapper classes, so that a consumer in the game module writes one import
and gets the whole chapter.

Exported units:

- `GetGameVersion` — the build's version string.
- `GetGameDistribution` — the installed retail edition number.
- `GetGameID` — the numeric title id.

**Notes** — the aggregation is a convenience and a rebuild should not copy it: it is what
drags the vendor SDK's headers into every translation unit that touches multiplayer, and
it is the reason the header ends by undoing a dozen names the SDK's headers had
macro-defined out from under the engine (`min`, `max`, and the socket primitives). That
whole hazard is an artefact of C preprocessing and disappears with it.
