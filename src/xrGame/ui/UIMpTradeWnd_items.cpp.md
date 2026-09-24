# src/xrGame/ui/UIMpTradeWnd_items.cpp

> The buy screen's item table: one record per item in any state, the state machine that keeps
> buying and selling reversible, the convenience items the screen adds for you, and the saved
> loadouts.

**Needs** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`inventory_item.h`](../inventory_item.h.md) · [`Weapon.h`](../Weapon.h.md) · [`WeaponMagazinedWGrenade.h`](../WeaponMagazinedWGrenade.h.md) · [`Restrictions.h`](Restrictions.h.md)
**Used by** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T2: it instantiates real game objects to represent shop stock.

## Purpose

The screen's model. Two decisions dominate it.

**Every item on the screen is a real game object.** Picking a rifle off the shelf constructs an
actual weapon instance from its class identifier and section, because the screen needs to ask
it real questions — what ammunition does it take, can a scope go on it, what is attached right
now — and the only authority on those is the item itself. The cell widget's payload *is* that
object, and destroying the record destroys it.

**Nothing is deleted when you sell it.** Selling moves a record through a state, and the state
machine is what makes the screen fully reversible: cancel can put everything back because
everything is still there.

## The state machine

```text
                  buy                         sell
  shop  ────────────────────────▶  bought  ──────────▶  (back to shop)
  own   ────────────────────────▶  sold    ──────────▶  (back to own)
```

**Contract** — a state assignment is *not* a write. Each current state accepts only certain
requests and may translate them:

```text
FUNCTION set_state(requested)
  IF requested IS undefined THEN state <- undefined; RETURN     # the explicit reset
  CASE state OF
    undefined : state <- requested                              # first placement
    shop      : requested must be bought       -> state <- bought
    bought    : requested must be shop or sold -> state <- shop   # NOT sold
    own       : requested must be sold         -> state <- sold
    sold      : requested must be own or bought-> state <- own    # NOT bought
```

**Invariants** — the two translating cases are the whole point. Selling something you bought
this session returns it to the *shelf*, not to a sold pile; re-buying something you sold
returns it to *own*, not to bought. So a buy followed by a sell is exactly a no-op, and the
player's money and the shop's stock both return to where they were. A rebuild that stores the
requested state verbatim breaks cancel.

The undefined reset exists so that the few places that must force a state — restoring a failed
swap, returning everything to the shelf on close — can do so through the same door.

## The item table

**Contract** — a flat list of every record. Lookups are linear and by (section, state), by
(section, state, attachment mask), by state alone, or by cell widget. Counting uses the same
predicates. A lookup by widget that fails is a fatal inconsistency; the others return nothing.

**Notes** — the table is flat and searched linearly at a scale of a few hundred records, which
is fine. What matters to a rebuild is the *key space*: an item is identified by its section
**and** its state **and**, for weapons, its attachment mask, because the screen routinely holds
several records for the same section in different states at once.

## Convenience items

**Contract** — four of the player's lists — the two ammunition lists, the medical list and the
grenade list — are filled automatically rather than by the player. Refreshing them deletes every
automatic item in them and rebuilds:

- the medical and grenade lists get one of *every* item the whole store sells that routes to
  that list;
- each ammunition list gets one of every ammunition type the weapon in the adjacent slot
  accepts — plus, for a rifle with a grenade launcher attached, its launcher ammunition.

An automatic item is bought at zero cost and is excluded from every loadout.

```text
FUNCTION refresh_convenience_items()
  delete every automatic item in the four lists
  FOR EACH list IN [medical, grenades]
    FOR EACH item section sold anywhere in the store that routes to this list
      create it, mark it automatic, buy it
  FOR EACH (ammo_list, weapon_list) IN [(pistol_ammo, pistol), (rifle_ammo, rifle)]
    IF weapon_list is empty THEN CONTINUE
    weapon <- the single item in weapon_list
    FOR EACH ammunition type the weapon accepts (and its launcher's, if attached)
      IF the store sells it THEN create it, mark it automatic, buy it
```

**Notes** — this is the screen's most player-visible convenience and its least obvious mechanic:
those four lists always show everything you *could* take, at no cost, and what you actually
take is decided at accept time. The zero cost is why an automatic item bypasses the buy checks
entirely. A rebuild that renders them as a menu rather than as bought items must reproduce the
consequence: a loadout never records them, and selling one is a real sale.

## `UpdateCorrespondingItemsForList` — keeping ammunition with its weapon

**Contract** — when a weapon slot changes, the adjacent ammunition list must be re-derived.
Everything currently in it is moved to the bag; the automatic items are rebuilt for the new
weapon; then, for each item in the bag that the new weapon *needs*, one is pulled back into the
ammunition list; and finally anything that came out of the old list and is now needed by
nothing is sold.

**Notes** — the rule is "ammunition follows its weapon". Swapping rifles turns your old rifle's
magazines into money rather than leaving them in the bag as dead weight, which is what a player
in a thirty-second buy window wants. The implementation walks the bag repeatedly and restarts
after each move, because moving an item renumbers the list — a rebuild that iterates a snapshot
gets the same result more simply.

## Loadouts

**Contract** — storing a loadout records, for each distinct (section, attachment mask) among the
player's non-automatic items, how many are held and which attachments by name. The result is
sorted into a fixed *priority* order — outfit, medical, grenades, rifle, pistol, ammunition,
everything else — so that applying it later places the important things first while money lasts.
Applying a loadout sells everything, rebuilds the convenience items, and then buys each entry up
to its recorded count, attaching the recorded attachments; anything that cannot be afforded is
simply not bought.

```text
FUNCTION store_preset(slot, announce, only_purchasable, refresh_helpers)
  IF refresh_helpers THEN delete the automatic items    # so they are never recorded
  IF announce THEN show "stored to slot N"
  entries <- empty
  FOR EACH record IN item_table WHERE state IS bought OR own
    skip automatic items
    key <- (record.section, its attachment mask)
    skip if key already recorded
    count <- number of bought + own records with that key
    skip if zero
    IF only_purchasable AND the store does not sell that section THEN skip
    entries.append(key, count, the attachment section names)
  sort entries by list priority, descending
  IF refresh_helpers THEN rebuild the automatic items

FUNCTION apply_preset(slot)
  sell everything;  rebuild the automatic items
  FOR EACH entry IN preset[slot]
    FOR i FROM (how many the player already owns) TO entry.count - 1
      item <- create(entry.section)
      IF NOT buy(item) THEN destroy(item); CONTINUE   # out of money or restricted: skip it
      FOR EACH attachment named in the entry
        addon <- create(that section)
        IF NOT buy(addon, attach to item) THEN destroy(addon)
```

**Notes**

- **The priority order is not the list order.** Outfit outranks everything, then medical, then
  grenades, then rifle, then pistol, then the ammunition lists, then the bag. So a loadout the
  player can no longer afford degrades by dropping ammunition and sidearms first, which is
  deliberate — armour and a primary weapon are what keep you alive. The numbers are a fixed
  table indexed by the item's destination list.
- The *only purchasable* filter exists because a loadout can be stored from a state the player
  was given rather than bought — the arrival loadout — and such a loadout must stay applicable
  even though the store does not sell those items.
- Applying starts from what the player already owns rather than from zero, so applying the same
  loadout twice buys nothing the second time.

## `SellAll`, `ResetToOrigin`, `CleanUserItems`

**Contract** — *sell all* repeatedly sells the first bought item, then the first owned item,
until neither exists. *Reset to origin* sells every bought item and then re-buys every sold one,
which is exactly the inverse of the session. *Clean user items* forcibly returns every record in
any player state to the shelf: it detaches attachments, forces each record through the undefined
reset to the shelf state, clears its owning list, and finally empties all nine player lists.

**Notes** — the three differ in who they answer to. Reset-to-origin is the player pressing
*reset*; clean-user-items is the screen closing and is allowed to break the state machine's
rules because nothing will observe the result.

## The shelf tint overlay

**Contract** — a per-cell draw hook on every shelf item. It writes the item's keyboard
accelerator as a digit at the cell's corner, then tints the cell: one colour when the player's
rank forbids the item, a second when the player cannot afford it, and the normal colour
otherwise. Rank is checked before money, so a rank-locked item never reads as merely expensive.

**Notes** — the accelerator digit is recovered by subtracting the first digit key's scancode
from the stored accelerator, with a special case remapping one particular value. That special
case does not correspond to any reachable accelerator and its intent is not recoverable from
the source; it is almost certainly a leftover from an earlier key encoding.
