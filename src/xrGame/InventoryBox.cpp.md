# src/xrGame/InventoryBox.cpp

> A container placed in the world: a stash, crate or locker that holds items without being able to use them.

**Needs** — [`InventoryBox.h`](InventoryBox.h.md) · [`GameObject.h`](GameObject.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`InventoryBox.h`](InventoryBox.h.md); callers name that, not this file.
**Tier floor** — T3: a list of identifiers and four event cases

## Purpose

A world object that owns items. It is deliberately **not** an inventory: it has no slots, no
belt, no weight and no active item, because a box never wields anything. What it needs is
the ability to receive and release ownership of items, to hide them while they are inside,
and to be opened, locked or emptied by script.

That reduction is the whole file. Ownership arrives and leaves as network events, so the
same code serves a stash the player fills, a crate a script populates, and a trade.

## State

```text
RECORD InventoryBox
  items     : list<entity id>   # what is inside, by identity not by handle
  in_use    : bool              # a screen is currently showing this box
  can_take  : bool              # items may be removed
  closed    : bool              # the box refuses to open at all
  tip_text  : text              # string-table key for the "use" prompt
```

**Invariants** — contents are held **by entity identifier**, never by object handle. A box
outlives the objects inside it across a level change and an alife promotion, and an
identifier survives both where a handle does not. Every read resolves the identifier through
the level's object registry at the moment of use.

`can_take` and `closed` are separate: a closed box cannot be opened; an open box with
`can_take` false shows its contents but yields nothing. Scripted story stashes use the
second to display a reward before it is earned.

## `net_Spawn`

**Contract** — brings the box online from its server record. After the base spawn it becomes
visible and enabled, takes a default prompt, and then — if the record is an alife box
record — overrides the takeable flag, the closed flag and the prompt from it. A box whose
record is of another kind keeps the defaults.

**Invariants** — the default prompt is set *before* the record's override so a record
carrying no prompt still leaves a usable one.

## `OnEvent`

**Contract** — the four ownership events, all arriving as network messages so that the
single-player and multiplayer paths are identical. Base handling runs first in every case.

```text
FUNCTION on_event(packet, type)
  base.on_event(packet, type)
  SWITCH type
    trade_buy, ownership_take:
      id = packet.read entity id
      object = level.objects.find(id)
      items.append(id)
      object.set_parent(this)
      object.visible = false          # inside the box it is neither drawn
      object.enabled = false          # nor collided with
      IF a corpse-or-container search screen is open on THIS box
        tell it an item arrived

    trade_sell, ownership_reject:
      id = packet.read entity id
      object = level.objects.find(id)
      remove id from items
      just_before_destroy = packet has more AND reads true
      # a sold item and a doomed item both vanish; only a genuinely
      # released item gets a physics body and falls to the floor
      dont_create_shell = (type IS trade_sell) OR just_before_destroy
      object.set_parent(none, dont_create_shell)
      IF in_use
        fire the script callback "item taken from box"(this, object)
```

**Invariants** — the trailing "about to be destroyed" byte is *optional* in the message.
Readers must tolerate its absence, because the sender omits it on the ordinary release path.
A rebuild framing this message must keep the field optional or keep two message kinds.

**Notes** — the script callback fires only while a screen is open on the box, so a script
that empties a box programmatically gets no notification. It is also delivered through the
player rather than through the box, which means a box emptied while the player does not
exist would have nowhere to deliver it.

## `AddAvailableItems`

**Contract** — resolves every stored identifier to a live item and appends it to the
caller's list, for a screen to display. Every identifier is expected to resolve; a box
holding a stale identifier is an error, not an empty slot.

## `set_can_take` / `set_closed`

**Contract** — the script surface. Each sets its flag and then broadcasts the box's whole
status so other participants agree. Setting the closed state also replaces the prompt with a
caller-supplied reason — "locked", "welded shut" — falling back to the default prompt when
none is given, so a box never ends up with an empty prompt.

## `SE_update_status`

**Contract** — broadcasts the takeable flag, the closed flag and the prompt as one message.
All three travel together because they are read together by whatever decides whether a
player may interact.

## `UpdateCL` / `net_Destroy` / `net_Relcase`

**Contract** — pure delegation to the base object. They exist as overrides with no added
behaviour; a rebuild needs none of them.

**Notes** — in particular the box does **not** clear a destroyed object out of its contents
when the engine announces one is going away. The identifier stays in the list and the next
read of it fails. Nothing in the shipped games destroys an item while it is inside a box, so
the hole is unreachable — but a rebuild should drop the identifier here.
