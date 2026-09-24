# src/xrGame/actor_mp_server.cpp

> The server-side record of a networked player: the authoritative state the server relays, and the rule that a dead player stops being relayed.

**Needs** — [`actor_mp_server.h`](actor_mp_server.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`actor_mp_server.h`](actor_mp_server.h.md); callers name that, not this file.
**Tier floor** — T2: record bookkeeping around a frozen wire form

## Purpose

The server object for a networked player. In a match the server is a relay: it receives a
player's state from that player's own client, holds it as the authoritative record, and
sends it out to everyone else. The client object and this record therefore speak *the same
wire form* — the same holder type serializes both — and the record's job is to be the
point where that state is authoritative.

The send and receive halves are split into
[`actor_mp_server_export.cpp`](actor_mp_server_export.cpp.md) and
[`actor_mp_server_import.cpp`](actor_mp_server_import.cpp.md).

## State

```text
RECORD ServerPlayerRecord extends the alife player record
  state_holder     : wire state       # the last state received, ready to relay
  ready_to_update  : bool             # the held state is current; do not re-gather it
```

Invariant: the freshness flag is raised by anything that fills the held state, and the send
path only gathers a fresh state when it is down. It exists because the state normally
arrives from the client rather than being computed, so gathering it on the server would
overwrite the authoritative value with a stale local one.

## `Net_Relevant`

**Contract** — a player with no health is not relayed at all; otherwise the base record's
relevance rules apply.

**Invariants** — this is where the "dead players cost nothing" saving actually lands (the
client-side version of the same rule is disabled; see
[`actor_mp_client_export.cpp`](actor_mp_client_export.cpp.md)). A dead player's corpse is
a local simulation on each client, driven by the death event, not by continued updates.

## `on_death`

**Contract** — runs the base record's death handling, then gathers the player's state from
the record's own fields and stores it in the holder.

**Invariants** — this is the *last* state that will ever be relayed for this player, and
it is gathered explicitly because from this moment the relevance rule stops updates
arriving. Without it the final state — the position and pose the player died in — would be
whatever the last received update happened to hold. Clients need the death pose to place
the corpse.

## `STATE_Read` / `STATE_Write`

**Contract** — the full-state form, used when a player first becomes known to a client
rather than for periodic updates. Both delegate entirely to the base record; this class
adds only a debug trace of the health value.

**Notes** — the full-state form does *not* use the compact wire record. A player's spawn
record is sent once and carries the whole alife player record; the compact form is only
for the per-tick update. Two serializations of the same entity is normal in this engine
and the split is between *what an entity is* and *what an entity is doing*.
