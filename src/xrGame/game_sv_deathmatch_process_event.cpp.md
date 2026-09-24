# src/xrGame/game_sv_deathmatch_process_event.cpp

> Deathmatch's two extra server events: a player asked to be killed, and a player finished shopping.

**Needs** — [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md)
**Tier floor** — T3: a dispatch on an event type

## Purpose

One dispatch function in a file of its own; the split is a compilation artifact and a rebuild
folds it into [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md).

## State

`Stateless.`

## `OnEvent`

**Contract** — handles two events and passes the rest to the base.

- **Kill** — the player-initiated suicide, raised by the console command that kills you. The
  packet names a client by identifier; the server resolves it to a client record and kills
  that client's current body. An unknown identifier is ignored silently.
- **Buy finished** — the client closed the buy screen. Handled for **the sender**, not for
  any identifier in the packet.

**Invariants** — the two events read their subject from different places, and that is the
security-relevant decision. The suicide takes its target from the packet, so a client can ask
the server to kill *another* client; nothing checks that the named client is the sender. The
purchase takes its subject from the transport's sender identity, which cannot be forged. A
rebuild should take the subject from the sender in both cases.

The client record is resolved without a null check on the purchase path, so a purchase from a
client the server has already dropped dereferences nothing. The kill path does check. The
asymmetry is an oversight, not a decision.
