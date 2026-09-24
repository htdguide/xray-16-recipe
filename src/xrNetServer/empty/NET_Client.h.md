# src/xrNetServer/empty/NET_Client.h

> The client surface with the vendor's types struck out of it — the same declarations, minus
> everything that named the transport.

**Needs** — [`../NET_Common.h`](../NET_Common.h.md) · [`../NET_Shared.h`](../NET_Shared.h.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md)
**Used by** — [`NET_Client.cpp`](NET_Client.cpp.md)
**Tier floor** — T2: an interface declaration.

## Purpose

Declares the surface implemented in [`NET_Client.cpp`](NET_Client.cpp.md). It mirrors
[`../NET_Client.h`](../NET_Client.h.md) and the contracts are there; what this page records is
the difference, which is the useful part.

## What differs from the real header

Three transport handles are gone from the client's state — the endpoint and the two address
objects — and the per-session record shrinks to nothing but the session's name. That shrinkage
is the clearest statement in the module of how little of the client is actually about the
transport: **three fields**.

Two operations that are constant in the real header become ordinary ones here, and one query
loses its constancy, which is a mechanical consequence of the vendor's types disappearing
rather than a decision.

The message queue's lock bracket is named `LockQ`/`UnlockQ` instead of `Lock`/`Unlock`,
because without the vendor header there is no longer a name collision to avoid. Purely
incidental, and worth one sentence only because a reader comparing the two copies will trip
over it.

## Notes

A rebuild has one client header. See [`README.md`](README.md) for why this one exists and what
should replace it.
