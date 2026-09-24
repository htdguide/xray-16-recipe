# src/xrGame/xrServer_process_event_ownership.cpp

> Decides whether one entity may take another — the gate every pickup, purchase and forced hand-out passes through.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_svclient_validation.h`](xrServer_svclient_validation.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: overwrites a message type at a fixed byte offset in a received packet

## Purpose

Taking something is the act most worth cheating at, and this is the only place it happens. The
page is a list of six checks, each rejecting a different way the request could be wrong or
hostile, and the order they run in.

## State

`Stateless.`

## `Process_event_ownership`

**Contract** — attach an item to a new parent. Reads the item's identifier from the payload; the
parent is the event's destination. Runs six checks, then asks the game rules, then rebuilds the
parent/child links and broadcasts. Silently does nothing on any refusal — a refused pickup is not
an error, it is a race lost.

```text
FUNCTION process_ownership(packet, sender, parent_id, forced)
  item_id := packet.read_int(16-bit)
  parent  := entities[parent_id]
  item    := entities[item_id]

  # 1. both must exist
  IF parent is absent THEN log and RETURN
  IF item is absent THEN RETURN                  # common and unremarkable; not logged

  # 2. both must be live on the simulating client, not merely present on the server
  IF NOT valid_on_simulating_client(parent_id) THEN log and RETURN
  IF NOT valid_on_simulating_client(item_id)   THEN log and RETURN

  # 3. the item must be free
  IF item already has a parent THEN RETURN

  # 4. authority: only the server, or the client that will own the result
  IF sender is neither the host's client nor the parent's owning client THEN RETURN

  # 5. the dead may not loot, outside single player
  IF parent is a creature AND it is dead AND the game is multiplayer THEN RETURN

  # 6. the rules decide
  IF NOT rules.on_touch(parent_id, item_id, forced) THEN RETURN

  IF the item and the parent are simulated by different clients
    migrate the item's simulation to the parent's client

  item.parent := parent_id
  append item_id to parent.children

  IF forced
    rewrite the event type in the packet to a plain "take"
  broadcast to everyone, including the sender
```

**Invariants** — **check 3 is the race guard and check 4 is the authority guard, and they are
different things.** Two clients reaching for the same item: the first wins on check 3, the second
finds a parent already set. A client claiming an item for somebody *else's* inventory: refused by
check 4 regardless of the race.

Check 2 is subtler. An entity can exist in the server's table and not yet be live on the client
that simulates it — spawned but not yet constructed there. Attaching to such a parent produces an
item that exists nowhere the player can see. See
[`xrServer_svclient_validation.cpp`](xrServer_svclient_validation.cpp.md).

The dead-cannot-loot rule is **suspended in single player**, where the player is expected to loot
corpses and the "parent" in question is the *looter*, not the corpse. The multiplayer reading is
that a dying player should not grab something on their way down.

**Notes** — **the forced variant rewrites the event type in the received packet, in place, at a
fixed byte offset.** The event arrives as "forced take" and is rebroadcast as a plain "take", so
clients apply it without re-running their own refusal logic. The source apologizes for the
technique in a comment, and it deserves the apology: the offset is a hard-coded constant that
silently encodes the event header's layout. A rebuild should compose the outgoing event rather
than patching the incoming one.

A purchase and a pickup are the same operation here — the payment is handled by the rules layer's
touch hook, which is where a transaction can refuse. That is why buying and taking share a code
path at all.
