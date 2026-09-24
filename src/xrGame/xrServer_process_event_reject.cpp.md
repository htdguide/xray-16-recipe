# src/xrGame/xrServer_process_event_reject.cpp

> Detaches an item from its owner — the inverse of taking, and the step that has to succeed before anything can be destroyed or handed on.

**Needs** — [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: mutates the ownership tree and re-emits the received packet

## Purpose

Dropping, selling and launching all reduce to the same act: sever the parent/child link. This is
also the step that [`xrServer_sls_clear.cpp`](xrServer_sls_clear.cpp.md) and
[`xrServer_process_event_destroy.cpp`](xrServer_process_event_destroy.cpp.md) call internally to
make an entity a root before destroying it, which is why it **returns a success flag** while its
ownership counterpart does not.

## State

`Stateless.`

## `Process_event_reject`

**Contract** — detach an item from a parent. Answers whether the detachment happened. Takes a flag
controlling whether the event is rebroadcast, so that an internal caller can detach quietly and
compose its own announcement.

```text
FUNCTION process_reject(packet, sender, time, parent_id, item_id, announce) -> bool
  parent := entities[parent_id]
  item   := entities[item_id]

  IF item is absent   THEN log and RETURN false
  IF parent is absent THEN log and RETURN false

  IF item_id is not in parent.children THEN warn and RETURN false   # not ours to release
  IF item has no parent                THEN log and RETURN false    # already free

  IF item.parent != parent_id
    log the disagreement — and proceed anyway                       # see Notes

  rules.on_detach(parent_id, item_id)
  item.parent := none
  remove item_id from parent.children

  IF announce THEN broadcast to everyone, including the sender
  RETURN true
```

**Invariants** — **the parent's child list and the item's parent field are checked separately**,
because they are two halves of the same invariant and either can be the one that is wrong. The
child-list check is the one that guards the actual removal; the parent-field check is diagnostic.

**Notes** — the case where the item names a *different* parent than the one releasing it is
logged with a message the source annotates as impossible, and then **proceeds to detach anyway**.
That is the wrong resolution: it clears a link the caller did not establish and leaves the real
parent's child list holding a stale entry. A rebuild should refuse. It survives because the
situation genuinely does not arise when the invariants hold, and because refusing would have
stalled the teardown paths that call this.

Notice there is **no authority check here** — unlike its ownership counterpart, any sender may
cause a detachment. The asymmetry is defensible for the original's threat model (taking is
valuable, dropping is not) and is a hole in a rebuild's: a hostile client can disarm another
player by rejecting their weapon. A rebuild should require the sender to own the parent.

The three event types that route here — dropping, selling, launching a rocket — differ only in
what the *rules layer* does in its detach hook. The server's own act is identical for all three.
