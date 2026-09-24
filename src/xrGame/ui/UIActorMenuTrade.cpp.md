# src/xrGame/ui/UIActorMenuTrade.cpp

> Trade mode: four lists arranged as two offers, a price that depends on which side owns the
> item, and a transaction that only commits when both sides can afford their half.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UITradeBar.h`](UITradeBar.h.md) · [`UIWeightBar.h`](UIWeightBar.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UICellItemFactory.h`](UICellItemFactory.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UITalkWnd.h`](UITalkWnd.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: list orchestration and integer arithmetic

## Purpose

Trade is a staging screen, not an immediate exchange. Both sides build an *offer* by moving
items into a dedicated list; nothing changes hands until the trade button is pressed, and at
that moment the whole offer either transfers or none of it does. This file holds the staging
rules, the pricing and the commit.

## The four lists

```text
  actor bag        <-> actor offer          |   partner offer <-> partner bag
```

Items move only along the arrows. The two offers never exchange with each other by dragging;
crossing the middle is what the trade button does. That constraint is expressed in the drop
table in [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md), and every move operation
here re-states it as an assertion.

## `InitTradeMode` / `DeInitTradeMode`

**Contract** — Entry shows the four lists, the two trade bars, the partner's panel and money,
the trade buttons and the partner weight bar; hides the inventory bag list. Then, in order:

1. tell the partner it is now trading — which is a *game* state change, not a UI one: a
   trading character behaves differently;
2. fill the actor's trade bag and the partner's bag;
3. take both sides' trade handles and start a trade session on each, **each pointed at the
   other**;
4. compute the initial prices.

Exit runs the reverse: stop both trade sessions, tell the partner it has stopped trading, hide
everything, retire any transient money message, and — if the talk screen is underneath —
**tell it to re-evaluate its question list**, because trading can change which conversation
topics are available.

**Invariants** — Both handles must be started; one-sided trade is not a state the pricing
understands. Stopping is guarded on each handle existing, because exit runs at destruction
even when entry never ran.

## `InitPartnerInventoryContents`

**Contract** — Refill the partner's bag from the partner's *available* items — the subset the
game exposes for trade, not the raw container — skipping anything already staged in the
partner's offer, sorted by descending footprint for the same packing reason as everywhere
else. Finally, record the partner inventory's modification counter, which is what the
per-frame update compares against to notice outside changes.

## `ToActorTrade` / `ToPartnerTrade` / `ToPartnerTradeBag`

**Contract** — The three staging moves.

`ToActorTrade` refuses outright when the partner would not accept the item (see
`CanMoveToPartner`), and refuses to stage anything coming from a quick slot. When the item was
not already loose in the ruck it is first moved there — a staged item must be in the bag, not
in a slot — and a to-bag event is emitted for that move.

`ToPartnerTrade` asks the *partner* whether the item may be staged at all, and declines with a
log line rather than failing when it may not. It re-prices afterwards.

`ToPartnerTradeBag` is the un-stage and has no rules: anything staged may be taken back.

**Invariants** — The asymmetry is the game rule. The actor may only offer what the partner
will buy; the partner may only offer what it is willing to sell; un-staging is always allowed
on both sides.

## `CanMoveToPartner` — the four refusals

**Contract** — Whether the actor may stage this item. All four conditions must hold:

```text
FUNCTION can_move_to_partner(item) -> bool
  IF item is untradeable                                 THEN RETURN false
  IF partner will not buy this item's section            THEN RETURN false
  IF item.condition < partner.minimum_buy_condition      THEN RETURN false

  # Weight: would the partner be over its carry limit after the whole
  # exchange, not just after this one item?
  actor_offer_weight   = total weight of the actor's offer
  partner_offer_weight = total weight of the partner's offer
  projected = partner.inventory_weight
            - partner_offer_weight        # what the partner is giving away
            + actor_offer_weight          # what the partner is taking on
            + item.weight
  RETURN projected <= partner.max_carry_weight
```

**Invariants** — The weight test projects the **whole pending exchange**, not the single item.
Adding one item at a time and checking each against the partner's *current* weight would let
the player stage an exchange that cannot complete.

Both parts of the test run on every colouring pass, which is why staged items get tinted red
as an offer grows.

## `ColorizeItem`

**Contract** — Tint a cell red when the partner will not take it, white otherwise. The one
visual channel that tells the player a refusal before they try it. Applied whenever a list is
filled or an item is moved, in trade mode only.

## `CalcItemsWeight` / `CalcItemsPrice`

**Contract** — Sum a list, **including every item merged into every stack**. The stacks are
the reason these are separate helpers rather than inline loops: a five-round stack of
ammunition is one cell and five prices.

Price is asked of a trade handle with a direction flag — buying or selling — because the same
item has two prices depending on which way it is moving.

## `UpdatePrices`

**Contract** — Re-derive everything the two trade bars show: the actor's money, the partner's
money, the total price and weight of each offer. The actor's offer is priced as *the partner
buying*; the partner's offer as *the partner selling*. Both prices come from the **partner's**
trade handle, never the actor's — the merchant sets both rates, which is the series' economy.

## `UpdateActor` / `UpdatePartnerBag`

**Contract** — `UpdateActor` writes the actor's money with the localized currency suffix into
both money widgets (they may be the same widget in one layout dialect), refreshes the weight
bar, and — a genuine side effect in a display routine — **forces the actor's active weapon to
re-read its ammunition count**, because trading ammunition away while holding the gun leaves
the on-screen count stale.

`UpdatePartnerBag` shows the partner's money, or a dash when the partner has infinite money (a
merchant), or nothing at all when the partner is an animal. Then updates the partner weight
bar from the bag's total.

## `OnBtnPerformTrade` and its two halves

**Contract** — The commit. Refuses when both offers are empty. Then:

```text
FUNCTION perform_trade()
  actor_price   = price of the actor's offer   (partner buying)
  partner_price = price of the partner's offer (partner selling)
  delta = actor_price - partner_price

  IF actor.money + delta >= 0
     AND partner.money - delta >= 0
     AND (actor_price >= 0 OR partner_price > 0)
    settle the money through the partner's trade handle
    transfer the actor's offer   -> the partner's bag   (partner buying)
    transfer the partner's offer -> the actor's bag     (partner selling)
  ELSE
    report which side could not afford it
  clear the selection; re-derive placement-dependent state
```

**Invariants** — **Both sides must be able to afford their half.** A merchant with finite
money can refuse a sale it cannot pay for, and the screen must say so rather than silently
producing negative money. The money is settled *before* the items move, so a failure to
transfer leaves money already paid — acceptable only because the transfer below cannot fail
for the actor's side.

`OnBtnPerformTradeBuy` and `OnBtnPerformTradeSell` are the one-directional variants for
layouts that offer separate buy and sell buttons: each zeroes the other side's price and
transfers only its own direction. The buy variant deliberately **does not check the partner's
money**, since the partner is receiving; the sell variant does not check the actor's.

## `TransferItems`

**Contract** — Move every cell of one offer list into a destination list, transferring
ownership through the trade handle as it goes.

```text
FUNCTION transfer_items(from, to, trade, buying)
  WHILE from is not empty
    cell = from.remove(from.item_at(0))
    trade.transfer(cell.item, buying)

    IF buying
      # The receiving side may refuse to carry it after all -- weight, or
      # a rule. The item has changed hands regardless; it simply does not
      # appear in the destination list.
      IF receiver accepts the item in its ruck THEN to.add(cell)
    ELSE
      to.add(cell)

  re-notify both sides' money so their displays refresh
```

**Invariants** — The loop consumes the source list, so an exception or an early exit leaves a
half-transferred offer. That is accepted because the whole affordability check happened before
the call.

The asymmetry — the buying direction may drop a cell, the selling direction may not — is
because the partner is the one with a carry limit. The item is not lost: ownership already
transferred, and it will appear the next time the partner's bag is refilled.

## `DonateCurrentItem`

**Contract** — Give an item to the partner outright, from the context menu during trade.
Requires the item to be in the actor's own bag; removes the cell, transfers ownership through
the partner's trade handle marked as a gift, and places the cell in the partner's offer list
so the player can see what they gave.

**Notes** — This is an addition to the original engine, not part of the shipped games' own
interface, and a rebuild targeting the retail games may omit it. It is described here because
it is reachable from the shipped context menu once present.
