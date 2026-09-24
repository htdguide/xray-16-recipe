# src/xrGame/trade2.cpp

> What an item costs, and what happens when it changes hands.

**Needs** — [`trade.h`](trade.h.md) · [`trade_parameters.h`](trade_parameters.h.md) · [`Actor.h`](Actor.h.md) · [`ai/trader/ai_trader.h`](ai/trader/ai_trader.h.md) · [`Artefact.h`](Artefact.h.md) · [`Inventory.h`](Inventory.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`Level.h`](Level.h.md) · [`game_object_space.h`](game_object_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`trade.h`](trade.h.md)
**Tier floor** — T2: a pricing formula and an ordered pair of network events.

## Purpose

Three things the session record in [`trade.cpp`](trade.cpp.md) does not hold: whether a trade
may start at all, what a price is, and the ordered sequence that moves an item and its money.
The split across two files is arbitrary.

The pricing formula is the economy of the whole game, and every one of its four factors is a
design decision that a rebuild must reproduce exactly or produce a different game.

## `CanTrade`

**Contract** — may a trade start right now? Scans for a partner, then applies three
conditions. Mutates the partner slot as a side effect — it selects the partner it found —
and clears it again on any failure.

```text
FUNCTION CanTrade() -> bool
  nearby := objects within 2 world units of me
  FOR EACH object IN nearby
    IF object IS an entity AND NOT object.alive
      RETURN false                       # a corpse nearby blocks trading outright
    IF SetPartner(object)
      BREAK
  IF no partner
    RETURN false

  d := distance(partner.position, my_position)
  IF d < 0.5 OR d > 4.5
    clear partner;  RETURN false

  face_to_face := angular difference between our two headings, in degrees
  IF face_to_face < 165 OR face_to_face > 195
    clear partner;  RETURN false

  RETURN true
```

**Invariants** — three conditions, each doing real work.

*The dead-body veto.* Encountering a corpse in the scan aborts the whole scan, not merely
that candidate. A trade cannot be opened standing over a body. This is the strongest of the
three and the only one that is a game-design statement rather than a usability one.

*The distance band, half a unit to four and a half.* Both ends matter: too close is a clipped
embrace, too far is out of conversational range. The scan radius is two units but the band
reaches to four and a half, so a partner found close can walk back a little without the
session becoming invalid.

*The facing window, 165 to 195 degrees.* The two parties must be turned toward each other
within fifteen degrees either side of exactly opposed. Facing the same way is not trading;
standing side by side is not trading. The window is what makes the player *address* a trader
rather than brush past one.

**Notes** — selecting the partner inside a question named "can" is a side effect, and the
error paths have to undo it explicitly. A rebuild should return the found partner instead.

## `GetItemPrice(item, buying, free)`

**Contract** — the price in money units of one item in the current session, from one's own
side's point of view. Zero for a gift, and zero for an item this side refuses to trade in the
named direction. Otherwise at least one and at most a million. Calls into script.

```text
FUNCTION GetItemPrice(item, buying, free) -> int
  IF free
    RETURN 0

  # 1. base cost
  IF item IS artefact AND my side IS the player AND partner IS a dedicated trader
    base := trader.artefact_price(item)      # traders value artefacts by their own table
  ELSE
    base := item.cost                        # the item's tuned cost

  # 2. condition: a ruined item is worth a tenth, a pristine one the full cost
  condition := (item.condition * 0.9 + 0.1) ^ 0.75

  # 3. relation: how the other party feels about me, mapped to 0..1
  attitude := relation_registry.attitude(me, partner)
  relation := (attitude IS unknown) ? 0 : clamp((attitude + 1000) / 2000, 0, 1)

  # 4. action: interpolate this side's friendly/hostile factor pair by the relation
  factors := my trade parameters for (buy or sell) and this item's section
  IF that action is disabled for this item's section
    RETURN 0                                 # I do not deal in this
  action := interpolate(factors.friend, factors.enemy, relation), clamped to their range
  IF action == 0
    RETURN 0

  price := floor(base * condition * action)

  # 5. script discount, by direction
  price := floor(price * script.trade_manager.discount(buying, my id))

  RETURN clamp(price, 1, 1000000)
```

**Invariants** — each factor is a separate decision.

*Condition* maps a ruined item to a tenth of its cost and a pristine one to the full cost —
the linear remap — and then applies a three-quarters power, which pulls the curve upward: an
item at half condition is worth about 63% rather than 55%. The effect is that mild wear
barely matters and only real ruin is punishing, which is what makes looting a firefight
worthwhile. Both the tenth and the exponent are tuned numbers with no derivation.

*Relation* maps an attitude running from -1000 to +1000 onto zero to one, with unknown
treated as maximally hostile. The window is the registry's own range, and the clamp catches
attitudes pushed beyond it by script.

*Action* is the only factor that comes from configuration rather than from the formula, and
it is where a rebuild will most easily go wrong. Each side declares, per item section, a pair
of factors: one for a friend and one for an enemy. The price factor is the interpolation
between them by the relation — and the interpolation is written to work **whichever way round
the pair is ordered**, so a configuration where the "friendly" factor is numerically larger
behaves as sensibly as one where it is smaller. The result is then clamped to the pair's own
range, so the interpolation can never escape the two authored numbers.

*A disabled section prices at zero*, which the caller reads as refusal. So "I do not buy
armour" and "I will give you this for nothing" are the same answer, distinguished only by the
gift flag. A rebuild is free to separate them but must keep zero meaning refusal for the
existing configuration.

*The direction asked of the configuration is the raw buying flag*, and the price is always
computed from **one's own side's** parameters. So a player buying from a trader is priced by
the *player's* buy factors, not the trader's sell factors. Whether that was intended is not
recoverable; the shipped configuration is tuned around it, and reversing it rescales every
price in the game.

*The script discount* is the modding hook, applied last and multiplicatively, looked up by
name in a well-known script module and passed the asking side's identity. A modder who does
not define it gets no discount rather than a failure.

*The final clamp* means nothing is ever free by arithmetic — the minimum is one, not zero —
so a worthless item still costs something, and the gift path is the only way to move an item
for nothing.

**Notes** — a fifth factor, a per-partner scarcity multiplier, is present but disabled and
pinned to one. It would have made a trader pay more for what it is short of. A rebuild
should decide deliberately; the hook is authored into the trader interface.

## `TransferItem(item, buying, free)`

**Contract** — move one item between the two inventories and the money the other way. Emits
two network events in a fixed order and fires two pairs of notifications. Does not check
affordability — the caller has done that.

```text
FUNCTION TransferItem(item, buying, free)
  amount := GetItemPrice(item, buying, free)

  seller := buying ? partner : self         # who is giving the item up
  buyer  := buying ? self    : partner

  seller.on_before_sell(item)               # both sides are warned before anything moves
  buyer.on_before_buy(item)

  emit sell_event(from seller, item)        # authority removes it from the seller
  seller.money := seller.money + amount
  emit buy_event(to buyer, item)            # authority adds it to the buyer
  buyer.money := buyer.money - amount

  IF self IS a dedicated trader AND buying AND item IS artefact
    artefact_tasks_dirty := artefact_tasks_dirty OR trader.buy_artefact(item)

  IF either side IS the player
    fire script callback(item, direction, amount, the other party)
```

**Invariants** — the **order is the contract**, and it is the only part of this function that
a rebuild can get wrong invisibly.

*Both notifications fire before either event.* A side that wants to veto or to react — an
owner unequipping the item, a quest watching for it — sees a consistent world in which the
item has not yet moved.

*The item leaves the seller before it reaches the buyer.* Two separate authoritative events,
sell then buy. The item is briefly owned by nobody. That window is what makes the sequence
safe against the reverse ordering, in which an item would momentarily exist in two
inventories and any weight or slot limit would be evaluated against a world that never
happened.

*Money moves with the item, on each side, at the moment that side's event fires.* The seller
is paid as the item leaves; the buyer pays as it arrives. Money is adjusted locally and not
through an event, so the two sides of a network session each compute it — which works only
because the price is a pure function of state both sides have.

*Only the player's side gets a script callback*, and it is fired on the player regardless of
which side the player is. Its direction argument is true only when the player is the buyer
*and* is not the party running this call, which is the file's way of expressing "the player
received something" from either party's point of view.

The artefact bookkeeping is a trader-only, buying-only special case: a trader that buys an
artefact may have satisfied a quest, and the flag records that the server should be told —
though, as noted in [`trade.cpp`](trade.cpp.md), nothing ever reads the flag.

## `GetTradeInv(party)` and the partner accessors

**Contract** — which inventory a party trades out of — its carried inventory, for every kind
of party — and the three readers that reach the other side's session, inventory and owner.
Every one of them requires a live session.

**Notes** — the inventory helper exists only to name the disabled alternative beside it, in
which a dedicated trader traded out of a separate goods store rather than out of what it
carries. A rebuild that wants trader stock not to be lootable from the trader's corpse will
need that distinction back.
