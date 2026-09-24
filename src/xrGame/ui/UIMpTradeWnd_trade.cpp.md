# src/xrGame/ui/UIMpTradeWnd_trade.cpp

> Buying and selling: the three restrictions a purchase must pass, where a bought item lands,
> how the shelf replenishes, and the swap that makes dropping a second rifle into an occupied
> slot work.

**Needs** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`game_cl_deathmatch.h`](../game_cl_deathmatch.h.md) · [`game_cl_capture_the_artefact.h`](../game_cl_capture_the_artefact.h.md) · [`Restrictions.h`](Restrictions.h.md)
**Used by** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T3.

## Purpose

The economics. Every path that changes what the player holds goes through one of the three
functions here, which is why the file is worth reading before the drag-and-drop one.

## `CheckBuyPossibility`

**Contract** — answer whether an item section may be bought, testing only the checks the caller
asked for, and — unless silenced — writing a localized explanation to the screen's message line.
Three independent checks, short-circuiting in this order:

```text
FUNCTION can_buy(section, checks, silent) -> bool
  cost <- price_table.cost(section, player_rank)
  IF checks INCLUDES money AND player_money < cost THEN
    explain "cannot buy: not enough money, has X, needs Y";  RETURN false
  IF checks INCLUDES rank AND the restriction table forbids the section at this rank THEN
    explain "cannot buy: rank restriction, has <rank name>, needs <rank name>";  RETURN false
  IF checks INCLUDES group_count THEN
    group <- the section's restriction group
    IF (bought + own) in that group >= the group's cap THEN
      explain "cannot buy: count restriction, you already have N";  RETURN false
  RETURN true
```

**Notes** — the *group cap* is the loadout balance rule: not "one rifle" but "no more than *n*
of this class of thing", with classes and caps in a shipped table. Automatic items are excluded
from the count, which is why the four convenience lists do not consume the player's allowance.
A rebuild must keep group counting separate from list capacity — the lists cap *placement*, the
groups cap *ownership*.

**The price depends on the player's rank.** The price table is queried with the rank every time,
so the same item can cost differently for two players in the same match.

## `TryToBuyItem`

**Contract** — buy one record. Automatic items skip every check. Then, unless the caller asked to
ignore it, the *game mode* gets a veto — each multiplayer mode has its own opinion about what a
player may buy right now. On success: deduct the cost if money was being checked; set the record
to *own* or *bought* as the caller asked; detach the widget from wherever it was; and then
either attach it to a weapon as an attachment, or place it in the list that matches it. Finally
replenish the shelf and show the money change.

```text
FUNCTION buy(record, checks, parent_for_attachment) -> bool
  IF NOT (automatic OR can_buy(record.section, checks, loud)) THEN RETURN false
  IF checks DOES NOT include ignore_team AND the game mode forbids it THEN RETURN false
  cost <- automatic ? 0 : price_table.cost(record.section, rank)
  IF checks INCLUDES money THEN money <- money - cost
  record.state <- (checks INCLUDES mark_as_own) ? own : bought

  widget <- detach record.cell_item from its list, if any
  widget.tint <- normal
  IF the record is an attachment AND it can go onto parent_for_attachment
      (or, with no parent given, onto the rifle or the pistol) THEN
    attach it;  destroy the record            # the attachment is now part of the weapon
  ELSE
    target <- matched_list_for(record.section)
    target.place(widget);  clear its shelf overlay and accelerator
    re-derive the ammunition list for target
  replenish_shelf(record.section)
  IF this was a normal purchase with a cost THEN show "-cost"
  RETURN true
```

**Notes** — **an attachment stops existing once attached.** Its record is destroyed and the
weapon's own attachment bit mask becomes the only trace. That is why selling a weapon has to
re-create the attachment records, and why a loadout stores attachment *names* rather than
records. A rebuild that keeps attachments as separate objects must still present them as part of
the weapon in the lists, or the count restrictions stop matching.

## `BuyItemAction` — the swap

**Contract** — the entry point for "the player chose this item from the shelf". For an item whose
destination is one of the three single-occupancy slots that is already full, it performs a swap:
sell the incumbent without destroying it, buy the new one, and — if the purchase failed — put
the incumbent back exactly as it was, restoring its state by force. An incumbent identical to
the new item is refused outright. For everything else it is a plain buy.

```text
FUNCTION buy_item_action(record) -> bool
  slot <- the list that matches record.section
  IF slot is one of (pistol, rifle, outfit) AND slot is occupied THEN
    incumbent <- the item in slot
    IF incumbent is the same thing as record THEN RETURN false
    saved_state <- incumbent.state
    sell(incumbent, destroy: false)
    IF buy(record, normal) THEN
      IF incumbent ended up sold or undefined THEN destroy it
      RETURN true
    # rollback: force the incumbent back, ignoring the mode veto and its real state
    force incumbent.state to undefined
    buy(incumbent, check_money | ignore_team | mark_as_own)
    force incumbent.state to undefined, then to saved_state
    RETURN false
  RETURN buy(record, normal)
```

**Notes** — the rollback is the file's ugliest passage and its source says as much. It works
because the state machine's *undefined* reset is the one door that accepts anything, and because
re-buying with the money check but no mode veto cannot fail. The decision worth keeping is the
guarantee: **a failed swap leaves the screen exactly as it was, money included.** A rebuild with
a transaction — snapshot, attempt, restore — expresses the same thing without forcing states.

## `TryToSellItem`

**Contract** — sell one record. A non-automatic item first sells its three attachments, each
refunding its own price, and then refunds its own price at the current rank; an automatic item
refunds nothing. The widget is detached from its list, the record moves *sold*, and then: if the
shelf already holds a copy of that section, the record is destroyed; otherwise, if the *currently
browsed* category sells it, the record goes back onto the shelf with its accelerator and its
tint overlay restored. Finally the money change is shown and the ammunition list for the
vacated slot is re-derived.

**Notes**

- **The shelf holds one of each section, not a stock count.** Selling a second copy destroys it
  because the shelf already shows one. So the store is infinite and the shelf is a catalogue.
- Returning to the shelf is conditional on the *browsed* category, not on the store: selling a
  pistol while looking at the grenades destroys the record rather than putting it somewhere the
  player cannot see. Navigating back rebuilds the shelf from the category, so nothing is lost.
- The refund is the full current price, so buying and selling is free. With a rank change in
  between it would not be, but rank does not change while the screen is open.

## `RenewShopItem` — replenishing the shelf

**Contract** — if the browsed category sells a section and the shelf does not already show it,
create or reuse a shelf record for it, give it a keyboard accelerator derived from its position
within the category, attach the tint overlay, and place it on the shelf.

**Notes** — the accelerator is the digit keys, assigned by position in the leaf's item list, and
suppressed past the tenth item. Two places compute that cut-off and they disagree by one — one
allows a tenth item an accelerator and the other does not. The shipped categories are small
enough that it never shows.

## `ItemToSlot` / `ItemToBelt` / `ItemToRuck`

**Contract** — the three calls the game layer uses to tell the screen what the player already
owns. Each creates a record in the *own* state, applies an attachment mask where one is given,
and places it in the matching list. The slot form additionally re-derives that slot's ammunition
list. Each asserts the section is one the price table knows.

**Notes** — these bypass every check and every cost, which is correct: the player is not buying
these, they arrived with them. The distinction between the three is which list the section
matches, not which call was used — belt and ruck differ only in that neither re-derives
ammunition.
