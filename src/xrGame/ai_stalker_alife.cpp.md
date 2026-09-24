# src/xrGame/ai_stalker_alife.cpp

> How a stalker decides what to carry: a virtual shopping pass that re-equips a character from everything it can reach, the rule that keeps it from hoarding two of the same weapon class, and the trade filter that decides what it will part with.

**Needs** — [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`Inventory.h`](Inventory.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`Weapon.h`](Weapon.h.md) · [`Grenade.h`](Grenade.h.md) · [`PDA.h`](PDA.h.md) · [`medkit.h`](medkit.h.md) · [`eatable_item.h`](eatable_item.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`ef_primary.h`](ef_primary.h.md) · [`ef_pattern.h`](ef_pattern.h.md) · [`trade_parameters.h`](trade_parameters.h.md) · [`clsid_game.h`](../xrServerEntities/clsid_game.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: scoring and selection over small lists; the only wire-format contact is the event messages it emits.

## Purpose

A stalker's inventory is not authored item by item. The spawn record gives it a rough
loadout and a purse; this file decides which of the things within its reach it should
actually be carrying, by *scoring* candidates with the engine's tuned evaluation functions
and buying the best it can afford. The same machinery, run against its own inventory,
answers the trade screen's question "will he sell me this?" — an item he would not buy for
himself is an item he is willing to sell.

It is a separate file from the stalker's main implementation because it is the character's
economic layer: trade, equipment choice and the single-item rule, none of which touch
perception, movement or combat.

## State

```text
RECORD TradeItem                # one candidate in a shopping pass
  item          : reference to an inventory item
  old_owner_id  : entity identifier    # who has it now
  new_owner_id  : entity identifier    # who will have it after the pass

# Ordered by the item's entity identifier, and compared for equality against a bare
# identifier, so the candidate list can be sorted once and searched by identifier.

# Carried on the stalker:
  temp_items         : list<TradeItem>   # the current shopping pass's candidates
  total_money        : int               # purse plus the notional value of everything
                                         #   sellable it already carries
  current_trader     : optional reference to the counterparty
  sell_info_actuality: bool              # is temp_items in step with the inventory?
  can_select_items   : bool              # may this character re-equip at all

# Invariant: after a pass, every tradable item in the inventory appears in temp_items
#   exactly once, and its new_owner_id is either this stalker (keep) or not (sell).
# Invariant: total_money is a *virtual* budget. Nothing is paid until a transfer is
#   actually issued; the pass only decides.
```

## `update_sell_info` — the shopping pass

**Contract** — Rebuilds the candidate list and re-decides the whole loadout. Costs a scan
of the inventory plus one scored pass per equipment category. Mutates only the stalker's
own decision state; no item changes hands.

```text
FUNCTION update_sell_info()
  actuality = true
  temp_items = empty
  current_trader = none
  total_money = purse
  FOR EACH tradable item in the inventory
    append it as a candidate, owner = self, new owner = "nobody"
    total_money = total_money + its cost      # everything sellable is spendable
  sort candidates by entity identifier
  select_items()                              # decide what to keep
  FOR EACH non-tradable item in the inventory
    append it as a candidate with new owner = self    # never for sale
```

**Invariants** — Starting every candidate's new owner at *nobody* is what makes the pass
mean "sell unless chosen": `select_items` only ever assigns ownership to the stalker, so
anything it does not pick stays marked as not-kept. The non-tradable items are appended
*after* the decision precisely so they cannot be sold — they enter the list already
assigned to their owner.

The budget counting every sellable item's cost as money is the trick that lets the pass
re-equip a character who owns a good rifle and no cash: he can "afford" a better one by
notionally selling what he has.

**Notes** — The early-out on the actuality flag is commented out, so the pass runs on every
query. The flag is still set, so the intent was caching; something made it wrong and the
fix was to disable it. A rebuild wanting the cache must invalidate it on every inventory
change and on every money change, in both directions.

## `tradable_item`

**Contract** — Decides whether an item may enter a shopping pass at all. Three gates: it
must be marked useful to a non-player character (so the player's quest junk is excluded);
if it is a personal data device, its *original* owner must not be the current holder — a
character will never trade away his own device, because its identity is evidence in the
story; and the item's configuration section must be enabled for selling in the character's
trade parameters.

## `select_items`

**Contract** — The whole re-equipment decision, in a fixed order. Does nothing at all for a
character whose re-equipment is disabled.

```text
FUNCTION select_items()
  IF NOT can_select_items THEN RETURN
  choose_food()                                  # disabled by design
  choose_weapon(Knife)
  choose_weapon(Secondary)                       # sidearm
  choose_weapon(Primary)                         # rifle
  choose_weapon(Grenade)
  choose_medikit()                               # disabled by design
  choose_detector()
  choose_equipment()                             # disabled by design
```

**Invariants** — The order is a spending priority: each category commits budget before the
next is considered, so a knife is bought before a rifle and a rifle before grenades.
Reordering changes what a poor stalker ends up carrying.

**Notes** — Three of the eight categories return immediately, each with a comment saying
the character may not change that category "due to the game design". Food, medical supplies
and armour are authored per character and must stay authored: a stalker who re-bought his
own suit would drift away from the loadout the level designer gave him. The empty functions
are kept rather than deleted so the priority order stays readable.

## `choose_weapon`

**Contract** — For one weapon category, scores every affordable candidate of that category
with the matching tuned evaluation function and buys the single best. Buying a weapon is
immediately followed by buying ammunition for it.

```text
FUNCTION choose_weapon(category)
  best = none; best_value = below any real score
  point the evaluation context at this stalker
  FOR EACH candidate
    IF its cost exceeds the remaining budget THEN CONTINUE
    point the evaluation context at this candidate
    IF the candidate's weapon class is not in this category THEN CONTINUE
    value = the category's evaluation function applied to the context
    IF the candidate is currently in an equipped slot
      value = value + 10          # strong preference for what he is already holding
    IF value > best_value THEN remember it
  IF best EXISTS
    buy it virtually
    attach_available_ammo(it)
```

**Invariants** — The four categories are selected by the item's weapon-class number, and
the mapping is data: knife is one class, grenade another, sidearm a third, and *six*
distinct classes count as a primary weapon. Those numbers come from the configuration and
are not derivable from anything in this file — a rebuild must carry the same table.

The three weapon categories score with three different evaluation functions (a general
item value for knives and grenades, a small-weapon value for sidearms, a main-weapon value
for rifles), so the scales are not comparable across categories — which is fine, since the
comparison is always within one.

**Notes** — The plus-ten bonus for an already-equipped item is the *hysteresis* that stops
a stalker from re-equipping every time the pass runs and two weapons score within noise of
each other. Ten is large relative to the evaluation functions' output range, so in practice
an equipped weapon is replaced only by a clearly better one.

## `attach_available_ammo`

**Contract** — Buys ammunition matching a just-bought weapon, affordable first, stopping
after one box. Does nothing for a weapon with no ammunition types.

**Notes** — The limit is one box per weapon, named as a constant. The engine does not model
a stalker running dry over a long fight through this path; ammunition is topped up by other
means.

## `choose_detector`

**Contract** — The same shape as `choose_weapon` but with no category filter — every
affordable anomaly detector is scored by the detector evaluation function and the best is
bought. No equipped-item bonus, so a stalker *will* swap detectors on a tie.

## `buy_item_virtual`

**Contract** — Marks a candidate as kept, deducts its cost from the virtual budget, and —
if a counterparty is set — credits that counterparty with the price. Nothing moves; the
actual transfer happens later, if at all.

## `transfer_item`

**Contract** — Actually moves one item between two owners, by emitting two network events
in order: a *sell* addressed to the old owner and a *buy* addressed to the new one, each
carrying the item's entity identifier.

**Invariants** — The order is load-bearing and the pair is not atomic: the item is removed
from the seller before it is added to the buyer, so a listener observing only one of the
two sees the item nowhere rather than in two places. That is the safer of the two failure
modes, and it is why the sell comes first.

Everything in trade goes through events rather than direct inventory mutation because the
authoritative record lives on the server side; the same code path must work when the two
owners are on different machines.

## `can_sell`

**Contract** — Answers whether a specific item is for sale. A character configured as a
dedicated trader sells anything tradable, with no shopping pass at all — traders do not
re-equip themselves. Everyone else runs the pass and answers "yes" exactly when the pass
did not keep the item. Asserts the item is in the candidate list; an item absent from it
means the inventory changed under the pass.

## `AllowItemToTrade`

**Contract** — The trade screen's gate. A *dead* character's inventory is governed not by
what he would sell but by what his trade parameters allow to be *shown* — looting a corpse
is a different permission from trading with its owner. A living one defers to `can_sell`.

## The single-item rule

A stalker must not accumulate two rifles. The rule is enforced when an item is taken, not
when it is offered, and it decides which of the two to drop rather than refusing the pickup.

### `non_conflicted`

**Contract** — Two items conflict only when both are weapons of the same weapon class. An
item compared against itself, a non-weapon, or a weapon of a different class never
conflicts.

### `enough_ammo`

**Contract** — True when the inventory holds at least one box of any ammunition the weapon
accepts. The threshold is a named constant of one: the question is "can he use it at all",
not "is he well supplied".

### `conflicted`

**Contract** — Given an item already held and a newly taken weapon, decides whether the
held item wins. The tie-breaks are applied in a fixed order and the first that separates
them decides.

```text
FUNCTION conflicted(held, new, new_has_ammo, new_rank) -> bool   # true = held wins
  IF they do not conflict THEN RETURN false
  held_has_ammo = enough_ammo(held)
  IF held_has_ammo AND NOT new_has_ammo THEN RETURN true     # 1. usable beats better
  IF NOT held_has_ammo AND new_has_ammo THEN RETURN false
  IF their conditions differ by more than 5 percent
    RETURN held.condition >= new.condition                   # 2. the less worn one
  IF their weapon classes differ
    RETURN held.cost >= new.cost                             # 3. the dearer one
  IF their ranks differ
    RETURN held.rank >= new.rank                             # 4. the higher-ranked one
  RETURN true                                                # 5. indistinguishable:
                                                             #    keep what he has
```

**Invariants** — Ammunition outranks quality: a working pistol beats an empty rifle. The
five-percent condition band is a deadband, not a rounding tolerance — without it, two
identical weapons differing by wear noise would flip the decision every time one was fired.
The final *keep what he has* makes the rule stable under repeated evaluation.

The class comparison inside `conflicted` can only fire when the two are the *same* class by
the conflict test above, so the cost tie-break is unreachable as written. It looks like a
guard left over from a wider definition of conflict.

### `can_take`

**Contract** — Whether this stalker should pick a loose item up at all: only weapons are
considered, and only if nothing already carried would win the conflict against it. A
non-weapon is always refused through this path — item pickup for everything else is
decided elsewhere.

### `on_after_take` and `update_conflicted`

**Contract** — Run after an item is actually taken. Skipped for a dead character and for
any character whose configuration disables the single-item rule — which is a per-character
switch, defaulting on, so that a designated pack mule or a trader can carry duplicates.
For each conflicting item already held: destroy the ammunition only that item could use,
then mark it to be dropped by hand rather than automatically.

```text
FUNCTION remove_personal_only_ammo(losing_weapon)
  FOR EACH ammunition type the losing weapon accepts
    IF any *other* weapon in the inventory also accepts that type
      CONTINUE                  # still useful; keep it
    destroy every item in the inventory of that ammunition type
```

**Invariants** — The ammunition sweep must run *before* the weapon is marked for dropping,
or the inventory being scanned no longer contains the weapon whose needs define "personal
only". Destroying rather than dropping is deliberate: a dropped magazine for a weapon
nobody has is litter the player would have to sort through, and the engine has a standing
interest in not growing the world's object count.

The manual-drop mark rather than an immediate drop leaves the *timing* to the animation
layer, so the character is seen putting the weapon down instead of it vanishing.
