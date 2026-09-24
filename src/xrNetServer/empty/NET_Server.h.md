# src/xrNetServer/empty/NET_Server.h

> The server surface with the vendor's types struck out of it — the same declarations, minus
> the two transport handles and one address-conversion overload.

**Needs** — [`../NET_Common.h`](../NET_Common.h.md) · [`../NET_Shared.h`](../NET_Shared.h.md) · [`../NET_PlayersMonitor.h`](../NET_PlayersMonitor.h.md) · [`../ip_filter.h`](../ip_filter.h.md)
**Used by** — [`NET_Server.cpp`](NET_Server.cpp.md)
**Tier floor** — T2: an interface declaration.

## Purpose

Declares the surface implemented in [`NET_Server.cpp`](NET_Server.cpp.md). It mirrors
[`../NET_Server.h`](../NET_Server.h.md), including the whole eleven-operation extension
contract a game layer must satisfy; the contracts are there.

## What differs from the real header

Two transport handles leave the server's state — the endpoint and the device address — and one
operation disappears entirely: the overload that converted a transport address object into the
engine's address record. That operation is the *entire* coupling between the server's address
handling and the vendor library, which is worth noticing: addresses are otherwise the engine's
own.

The peer's process identifier in the connecting client's identity blob is declared here as a
plain 32-bit integer where the real header uses the platform's process-identifier type. A
rebuild should treat it as an opaque integer supplied by the peer and trusted for nothing —
it is used only to recognize a second client from the same process.

The banned-list name accessor loses its constancy, and the ban expiry stays a wall-clock
timestamp. Both incidental.

## Notes

A rebuild has one server header. See [`README.md`](README.md) for why this one exists and what
should replace it.
