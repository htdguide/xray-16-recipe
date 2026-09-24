# src/xrGame/game_cl_capturetheartefact_buywnd.cpp

> The capture-the-artefact buy cycle: what a purchase looks like on the wire, why the client tracks money the server has not charged yet, and why the server is told when the menu merely opens.

**Needs** — [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`UIGameCTA.h`](UIGameCTA.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`Missile.h`](Missile.h.md) · [`eatable_item_object.h`](eatable_item_object.h.md) · [`game_base_menu_events.h`](game_base_menu_events.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: composes a purchase message and maintains one provisional balance

## Purpose

Four methods of the capture-the-artefact client mode, split into their own file because they
are the whole of the economy and nothing else in the mode touches it. They belong to the
class declared in
[`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md); the split is
organisational and a rebuild may merge them.

The economy's shape is worth stating once. A player shops **while dead**, between
reinforcement waves. The server charges him only when he spawns, so between confirming a
purchase and spawning there is a window in which the client knows about money the server
does not. That gap is the reason for a provisional-spend field, and it is why the three
apparently redundant menu events exist.

## State

`Stateless.` It reads and writes the mode's provisional spend and money indicator.

## `OnBuyMenu_Ok`

**Contract** — the player confirmed a purchase. Reads the chosen items and their net cost
from the menu, records the cost as the provisional spend if the player is dead, and sends one
event carrying the cost and the whole item list. During warm-up the cost is forced to zero.
A dead player additionally sends a menu-closed event. Marks the buy menu not ready, so it
must be rebuilt before it can be used again.

```text
FUNCTION on_buy_menu_ok()
  (items, cost) = buy_menu.purchase_items       # cost is a net delta, may be negative
  IF I am permanently dead
    buy_amount = warming_up ? 0 : cost
    update_money_indicator()

  event = new event(PLAYER_BUY_FINISHED, from my current entity)
  event.write_int32(warming_up ? 0 : cost)
  event.write_int16(count of items)
  FOR EACH (kind, count) IN items
    event.write_byte(kind) ; event.write_byte(count)
  send(event)

  IF I am permanently dead
    send event(PLAYER_BUYMENU_CLOSE, from my player)
  mark the buy menu not ready
```

**Invariants** — the warm-up rule is applied in **two** places, to the provisional spend and
to the wire value, and both must agree or the client and the server hold different balances
for the same player. A rebuild should compute the effective cost once.

**Notes** — an item is a byte pair: an **index into the menu's own item list** and a count.
That means the menu's ordering is part of the protocol — both ends must have built the same
menu from the same configuration section for the same team and rank, or the wrong items are
bought. It also caps a purchase at 255 kinds of at most 255 each. A rebuild sending item
section names instead pays a few bytes and loses the coupling entirely; the byte index is
the kind of decision that only makes sense when the configuration is frozen, which here it
is.

The cost is a *net* figure — the menu allows selling back, so it can be negative — which is
why it is sent explicitly rather than recomputed on the server from the item list.

The provisional spend is recorded only for a dead player. A living player buying at his base
is charged immediately, so there is no gap to bridge.

## `OnBuyMenu_Cancel` / `OnBuyMenuOpen`

**Contract** — a dead player opening or cancelling the buy menu tells the server, with a
menu-opened and a menu-closed event. Nothing is sent for a living player.

**Notes** — these exist because **the server holds a shopping player out of the next
reinforcement wave**. Without them a player would be respawned mid-purchase and lose the
loadout he was assembling. So the pair is not bookkeeping: it is a lock, expressed as two
events, whose scope is exactly the time the menu is open.

That also explains why confirming sends a *close* event of its own — the confirm path must
release the lock, and it does so with the same event rather than relying on the cancel path
running too.

## `LocalPlayerCanBuyItem`

**Contract** — may the local player buy this item section? The knife is always allowed;
everything else is the buy menu's decision, which accounts for money, rank and team.

**Notes** — the knife is named by a literal section and is free, so it is not in the priced
menu at all. Hardcoding it here rather than adding a free entry to every team's menu is the
kind of shortcut a rebuild should undo by giving the configuration a "starting equipment"
list.
