# src/xrGame/xrServer_process_event_activate.cpp

> Lets the game rules approve an artefact being switched on, and tells everyone — provided the thing is actually in somebody's hands.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a rules query and a rebroadcast

## Purpose

An artefact is activated by its holder and its effect is felt by everyone nearby, so activation
has to be authoritative even though the server does not simulate the effect. The server's whole
job here is to let the rules refuse, and to check one structural precondition the rules should not
have to.

## State

`Stateless.`

## `Process_event_activate`

**Contract** — approve and announce an activation. Both the holder and the item must exist —
their absence is a hard failure, not a refusal. Asks the rules; on approval, **verifies the item
actually has a parent** and refuses otherwise; then rebroadcasts to everyone including the sender,
unless the caller asked for silence.

```text
FUNCTION process_activate(packet, sender, time, holder_id, item_id, announce)
  holder := entities[holder_id]
  item   := entities[item_id]
  REQUIRE holder EXISTS                    # naming the two identifiers and the frame
  REQUIRE item EXISTS

  IF NOT rules.on_activate(holder_id, item_id) THEN RETURN

  IF item has no parent
    log "cannot activate an independent object"
    RETURN

  IF announce
    broadcast to everyone, including the sender
```

**Invariants** — **the parent check runs after the rules hook, not before**, so the rules see the
activation attempt even when it is structurally invalid. Whether that ordering is deliberate — a
rules layer that wants to count attempts — or accidental is not recoverable from the source. A
rebuild checking first would be defensible and would change what the rules observe.

**Notes** — missing entities are fatal here and merely logged in the ownership and reject paths.
The difference is that activation is only ever raised by a client that is holding the item and
looking at it, so a missing entity means the two sides' worlds have already diverged. Treating it
as fatal is how that divergence gets found; a rebuild on a live server should downgrade it to a
disconnect of the offending client rather than a process failure.

**No authority check.** Any sender may activate any artefact held by anyone. Same gap as in
[`xrServer_process_event_reject.cpp`](xrServer_process_event_reject.cpp.md), and the same remedy:
require the sender to own the holder.

The announcement uses a delivery mode distinct from the default — the same one the ownership and
reject paths use — which orders these three event families consistently with each other on the
wire. Since taking, dropping and activating are the three things that can happen to one item in
quick succession, their relative order must be preserved and a differently ordered channel would
break it.
