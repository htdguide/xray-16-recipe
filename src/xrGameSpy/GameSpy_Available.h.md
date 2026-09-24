# src/xrGameSpy/GameSpy_Available.h

> Declares the service reachability probe.

**Needs** — [`GameSpy_Available.cpp`](GameSpy_Available.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Used by** — [`xrGameSpyServer.cpp`](../xrGame/xrGameSpyServer.cpp.md) · [`GameSpy_Available.cpp`](GameSpy_Available.cpp.md) · [`GameSpy_Full.cpp`](GameSpy_Full.cpp.md) · [`xrGameSpy.h`](xrGameSpy.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`GameSpy_Available.cpp`](GameSpy_Available.cpp.md).

- `CheckAvailableServices` — blocking three-valued probe of the service's liveness for
  this title; yields a player-facing reason on failure.

**Notes** — the type is a class with one method and no state. A rebuild makes it a
function.
