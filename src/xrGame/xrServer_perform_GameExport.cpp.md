# src/xrGame/xrServer_perform_GameExport.cpp

> Pushes the whole game-rule state — scores, teams, round, limits — to every admitted client at once.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: serializes rule state into a reliable message per client

## Purpose

Most state reaches a client incrementally. Rule state does not: it is small, it is
client-specific, and getting it wrong is visible as a wrong score. So when it changes
structurally — a round ends, teams are rebalanced, a player joins — the rules layer raises a
flag and the *whole* state is re-sent to everybody. This file is that broadcast.

## State

`Stateless.`

## `Perform_game_export`

**Contract** — send every admitted client the full game-rule state, reliably. Skips clients not
yet accepted. **Clears the rules layer's resync flag** as its last act, so the flag is a
one-shot request.

```text
FUNCTION perform_game_export()
  FOR EACH client
    IF NOT client.accepted THEN CONTINUE
    packet := begin(SERVER_CONFIG_GAME)
    game.write_state_for(packet, client id)      # per-client: each sees its own view
    send reliably to client
  game.force_sync := false
```

**Invariants** — the state is written **per client**, not once and broadcast. Two clients on
opposing teams do not see the same rule state — one may not be told the other team's positions
— so a single broadcast would leak. That is why this is a loop with a serialize inside rather
than a broadcast of one buffer.

**Notes** — clearing the flag here rather than at the call site means any caller can request a
resync by setting it and the next frame performs exactly one. The server frame checks it twice
— once after the rules update and once after the outbound work — so a resync requested mid-frame
still goes out in that frame.

Resending everything rather than a delta is the right call for state this size, and it is the
reason rule state never desynchronizes the way entity state can.

## `Export_game_type`

**Contract** — tell one client the name of the game mode it has joined, before anything else
about the rules. Sent reliably.

**Notes** — the mode is sent **by name, as a string**, and the client instantiates its own
matching rules object from it through the same factory the server used. That is what keeps the
two sides' rule implementations paired without a version number: they agree on a name, and the
data-authenticity check has already established that both sides' data is the same. A rebuild
should keep the name rather than an ordinal, because the ordinal would change when a mode is
added.
