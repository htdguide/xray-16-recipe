# src/xrGame/xrClientsPool.h

> Declares the reconnect pool: parked client records, the identity test that reclaims one, and the expiry sweep.

**Needs** — [`xrClientsPool.cpp`](xrClientsPool.cpp.md) · [`xrServer.h`](xrServer.h.md)
**Used by** — [`xrClientsPool.cpp`](xrClientsPool.cpp.md) · [`xrServer.h`](xrServer.h.md)
**Tier floor** — T2: owns a vector of client records

## Purpose

Declares the surface implemented in [`xrClientsPool.cpp`](xrClientsPool.cpp.md).

Exported units:

- `Add(client)` — park a disconnecting client; takes ownership, or discards.
- `Get(new client)` — reclaim the pooled record for the same player, or nothing.
- `Clear` — free everything.
- `ClearExpiredClients` — private; the lazy sweep, run from `Get`.

## State

See [`xrClientsPool.cpp`](xrClientsPool.cpp.md).

**Notes** — the two predicates — "is this record expired" and "is this the same player" — are
declared as separate named things rather than inlined, and a rebuild should keep them named:
each encodes a policy decision (how long a player may be away, what makes two connections the
same person) that a server operator or a different game mode would plausibly want to change.
The expiry predicate carries its own copy of the current time and the window so that one
sweep uses one consistent clock reading across every record, which matters when the sweep
straddles a millisecond.
