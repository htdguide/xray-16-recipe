# src/xrGameSpy/GameSpy_QR2.h

> Declares the server-side advertiser and, more importantly, the callback set a server
> must supply for it.

**Needs** — [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`GameSpy_QR2_callbacks.cpp`](../xrGame/gamespy/GameSpy_QR2_callbacks.cpp.md) · [`GameSpy_QR2_callbacks.h`](../xrGame/gamespy/GameSpy_QR2_callbacks.h.md) · [`xrGameSpyServer.h`](../xrGame/xrGameSpyServer.h.md) · [`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md) · [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: a declaration surface.

## Purpose

Declares the surface implemented in [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md), where the
callback set — the actual contract between the matchmaking layer and the game's server —
is written out as a record.

- `SInitConfig` — the nine callbacks plus the server handle passed back to each. This is
  the interface a rebuild's game layer must satisfy, independent of any transport.
- `Init` — bind a query port, register with the master list, install the callbacks and the
  field names.
- `Think` — one poll per server frame: heartbeat and answer queries.
- `ShutDown` — deregister and close.
- `RegisterAdditionalKeys` — bind this game's field ids to wire names.
- `BufferAdd` · `BufferAdd_Int` · `KeyBufferAdd` — how a callback writes its answer.
- `RegisteredKey` — the wire name for a field id.
- `GetGameVersion` — the advertised build version.

**Notes**

The type has no state at all: every method reaches process-global registration. It is a
namespace, and a rebuild should make it one — or better, an object that actually owns the
advertiser handle, which is currently passed in and out as an opaque value by every
method that needs it.
