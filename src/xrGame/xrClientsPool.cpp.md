# src/xrGame/xrClientsPool.cpp

> Holds a disconnected multiplayer client's state for a while, so that a player who drops and comes straight back gets their score, team and inventory back rather than a fresh start.

**Needs** — [`xrClientsPool.h`](xrClientsPool.h.md) · [`xrServer.h`](xrServer.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrClientsPool.h`](xrClientsPool.h.md)
**Tier floor** — T2: a small vector of owned records with a timestamp sweep

## Purpose

A dropped connection in a multiplayer match is usually a network hiccup, not a departure.
Discarding the player's state immediately punishes them for their router; keeping it forever
leaks memory and lets a name be squatted. This is the compromise: the server parks the
disconnected client's record here, and either the same player reclaims it on reconnect or
it expires.

The *identity* question — is this new connection the same player — is the interesting part
and is answered here, not by the transport.

## State

```text
RECORD PooledClient
  client       : ref client record    # owned; the full server-side client state
  disconnected : int                  # global ms at which it was parked

RECORD ClientsPool
  clients : list<PooledClient>        # capacity reserved for the maximum player count
```

**Invariants** — the pool owns every record it holds and is the only thing that may free
one. A record handed back out by `Get` is no longer in the pool and ownership transfers to
the caller. The pool never exceeds the server's maximum player count in practice, because a
player cannot be both connected and pooled — but nothing enforces that, and a rebuild
should, since the reservation says the author assumed it.

## `Add`

**Contract** — park a disconnecting client's record, stamped with the current global time.
Takes ownership. **Discards the record outright, freeing it, in two cases**: when the client
carries no player state at all, and when the reconnect window is configured to zero.

```text
FUNCTION add(client)
  IF client has no player state THEN RETURN          # nothing worth keeping; leaked? see Notes
  IF reconnect_window_minutes == 0
    free(client)
    RETURN
  clients.append({ client, disconnected: now_ms() })
```

**Notes** — the first branch returns *without freeing*, while the second frees. A client with
no player state is one that disconnected before finishing its handshake, and the caller
still owns it in that case — so this is a two-caller-contract function, which a rebuild
should split into "park this" and "the caller decides". Getting it wrong leaks a client
record per aborted handshake.

A reconnect window of zero is the configured way to turn the feature off, and it is handled
by discarding at park time rather than by not calling — which means every caller can park
unconditionally.

## `Get`

**Contract** — given a *newly connected* client, find and remove the pooled record belonging
to the same player, if any. Sweeps expired records first, so a reconnect after the window
never matches. Transfers ownership of the returned record to the caller. Answers nothing
when there is no match.

**Invariants** — the match is removed from the pool, so a record is handed out at most once.

## Identity: `pooled_client_finder`

**Contract** — decides whether a new connection is the same player as a pooled one. Both
must carry player state. **Both the copy-protection key digest and the player name must
match.** Either differing is a non-match.

```text
FUNCTION same_player(new_client, pooled) -> bool
  IF either has no player state THEN RETURN false
  IF new_client.key_digest != pooled.client.key_digest THEN RETURN false
  RETURN new_client.name == pooled.client.name
```

**Notes** — **Two independent factors, and the reason is that neither alone is enough.** The
key digest identifies a *copy of the game*, not a person — two players sharing one key would
otherwise inherit each other's state. The name identifies a person but is chosen freely, so
alone it lets anyone claim a dropped player's score. Requiring both makes impersonation cost
the victim's key.

Neither factor involves the network address, deliberately: a player who reconnects from a
different address — a different route, a re-dialled link — is the case the whole feature
exists to serve.

A rebuild whose copy protection is absent has only the name, and should either accept that
reconnect is spoofable in a friendly game or issue a secret token at connect time and match
on that instead. The token is the better design and is what the key digest is standing in for.

## `ClearExpiredClients`

**Contract** — free and remove every record parked longer than the configured window. Called
from `Get`, so expiry is lazy: a pool that is never queried is never swept, and a record can
outlive its window until the next connection attempt.

**Invariants** — the window is configured in **minutes** and compared in milliseconds; the
conversion is here and the configured value is the human-facing unit.

**Notes** — lazy expiry means the memory is held until someone connects. In a match with no
further connections the pool holds its records to the end. That is acceptable because the
pool is bounded by the player count, and it is worth stating because a rebuild that sweeps on
a timer instead is *also* correct and slightly tidier.

## `Clear`

**Contract** — free every pooled record and empty the pool. Called at teardown and when the
server shuts a match down. Unconditional: a parked player loses their state regardless of the
window.
