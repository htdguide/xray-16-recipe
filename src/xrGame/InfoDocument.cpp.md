# src/xrGame/InfoDocument.cpp

> A document you can pick up: an inventory item whose only behaviour is to hand one information portion to whoever picks it up.

**Needs** — [`InfoDocument.h`](InfoDocument.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`PDA.h`](PDA.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one identifier and one event; everything else delegates

## Purpose

The game's unit of story knowledge is the **information portion** — a named fact a
character either holds or does not, which dialogue, quests and scripts all test against.
This item is the physical form of one: a flash drive, a note, a case file lying in the
world. Picking it up grants the portion.

That it is an *entity* rather than a script effect matters: the document can be carried,
dropped, traded and stolen, and the knowledge transfers with it every time it changes
hands.

## State

```text
RECORD InfoDocument
  info_id : text   # the information portion this document carries, from the spawn record
```

Invariant: the identifier comes from the **spawn record**, not from the configuration
section. Two documents of the same section carry different facts; the section supplies
the model and the weight, the record supplies the content.

## `net_Spawn`

**Contract** — spawn as an ordinary inventory item and take the information portion's
identifier from the server record, which is required to be a document record.

## `OnH_A_Chield`

**Contract** — on becoming someone's possession, if the new owner can hold an inventory
and the document names a portion, send an information-transfer event to that owner
naming this document as the sender, the portion as the payload, and *add* as the
operation.

**Invariants** — the grant is an **event**, not a direct call, so it travels the same path
as every other information transfer — through the server, into the recipient's known-facts
set, and out to the script callbacks and the PDA. A rebuild must not shortcut it, because
a great deal hangs off that one notification.

The transfer happens on the *after-becoming-a-child* hook, which is after the item is
actually in the inventory. Granting before would let a refused pickup still hand over the
knowledge.

```text
FUNCTION on_became_owned()
  base_on_became_owned()
  owner = the new parent, as an inventory owner
  IF owner is none OR this document names no portion THEN RETURN
  send an information-transfer event to owner:
    sender = this document's entity identifier
    payload = the portion identifier
    operation = add
```

**Notes** — the document is not consumed and the transfer is not marked as done. Dropping
and re-taking the same document re-sends the event; the recipient's known-facts set
absorbs the duplicate. A rebuild may keep that idempotence or track the grant, but must
not make a second grant an error.

## `Load`, `net_Destroy`, `shedule_Update`, `UpdateCL`, `OnH_B_Independent`

**Contract** — pure delegation to the generic inventory item. Present only to complete the
virtual surface; a rebuild deletes them.
