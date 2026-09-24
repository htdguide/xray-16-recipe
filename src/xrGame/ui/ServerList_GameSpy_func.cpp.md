# src/xrGame/ui/ServerList_GameSpy_func.cpp

> A hollowed-out translation unit that once held the server browser's vendor-service callbacks.

**Needs** — [`ServerList.h`](ServerList.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: no content

## Purpose

The file contains nothing but the include of the server-list declaration. The callbacks that
the matchmaking vendor's browser library invoked — server found, server updated, list
complete — were deleted when that service died, and the browser subscription in
[`ServerList.cpp`](ServerList.cpp.md) now goes through a wrapper instead.

The file is worth one line in the recipe because of what its emptiness records: the server
browser's data source is gone, and a rebuild that wants a working browser is designing a new
master-server protocol, not restoring this one.
