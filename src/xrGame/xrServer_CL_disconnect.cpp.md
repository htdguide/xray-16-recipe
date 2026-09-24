# src/xrGame/xrServer_CL_disconnect.cpp

> Decides what happens to everything a departing client was simulating: hand it to somebody else, or destroy the world.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`Level.h`](Level.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: walks the entity table and reassigns or destroys

## Purpose

A client leaving is not just a connection closing. Every entity it owned is now unsimulated, and
the server has two choices. The choice is made by counting clients, and the asymmetry between the
two branches is the whole file.

## State

`Stateless.`

## `OnCL_Disconnected`

**Contract** — handle one client's departure. Announces it to the rules layer, then either
migrates or destroys the entities the client owned. Returns immediately, doing nothing at all,
for a client that never got a player state.

```text
FUNCTION on_client_disconnected(client)
  IF client has no player state THEN RETURN      # never fully joined; nothing to unwind

  # tell the rules layer, deferred to the simulation thread
  event := (client id, player name, player's game identifier)
  game.queue_delayed_event(PLAYER_DISCONNECTED, event)

  IF more than one client remains AND the departing client is not the host
    FOR EACH entity owned by this client
      migrate it to the best other client, forcing a different owner
  ELSE
    destroy every entity in the table

  re-evaluate which client is the host      # clears it if the host just left
```

**Invariants** — **the destroy branch empties the entire entity table, not just this client's
entities.** That is correct for the case it serves — the last client leaving, or the host leaving
— because with no client left the world has nobody to simulate it and the session is over. It
would be catastrophic if the condition were wrong, and the condition is therefore the most
important line in the file.

**Notes** — the early return on a missing player state is the guard for a client that dropped
mid-handshake. Such a client owns nothing and has never been announced, so there is nothing to
unwind; announcing it would produce a disconnect message for a player who never appeared.

The disconnect event carries the player's **name and game identifier rather than a reference**,
because by the time the rules layer processes it the client record may be in the reconnect pool
or gone. Every deferred event in the chapter follows that rule: defer values, never references.

The migration branch passes the flag demanding a *different* owner, which is the evacuate case
described in [`xrServer_balance.cpp`](xrServer_balance.cpp.md) — and since that file always
answers "the host", migration here means "the host takes over". So in practice the two branches
are "the host takes over" and "the session ends", which is a much simpler policy than the code
shape suggests, and is the policy a rebuild should implement directly.

Re-checking which client is the host, last, is what clears the host reference when the host itself
is the one leaving. Skipping it leaves a reference to a record that the reconnect pool now owns.
