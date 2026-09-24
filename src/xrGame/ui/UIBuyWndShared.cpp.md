# src/xrGame/ui/UIBuyWndShared.cpp

> The multiplayer buy catalogue: every buyable item's five rank prices and which tab it
> appears under, in one name-sorted table whose index order is itself an identifier.

**Needs** — [`UIBuyWndShared.h`](UIBuyWndShared.h.md) · [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`Restrictions.h`](Restrictions.h.md)
**Used by** — [`UIBuyWndShared.h`](UIBuyWndShared.h.md)
**Tier floor** — T3: configuration parsing into a sorted table

## Purpose

The buy menu needs two facts about every item — what it costs and where it appears — and both
come from configuration. Keeping them in one sorted table means every consumer resolves an
item the same way, and means an item can be named by its *index* rather than by its section,
which is what the preset machinery and the network protocol both do.

## State

```text
RECORD CatalogueEntry
  slot_index : int (8-bit)   # which of the menu's lists this item appears in; 255 = unassigned
  cost       : int [5]       # one price per rank

RECORD Catalogue
  items : sorted map<section name, CatalogueEntry>   # ordered by lexicographic section name
```

Invariants:

- The map is **ordered by name**, and its iteration order is a stable identifier space. Index
  *n* means the same item on every machine that loaded the same configuration.
- Every entry must end with a real slot index. An item present in the price table but absent
  from the placement table is a configuration error, caught at dump time.
- An item may appear in exactly one slot's placement list; duplicates are fatal.

## `Load`

**Contract** — Build the table from two configuration sections: a price table whose every line
is `section = comma-separated prices`, and a placement table whose every line names one of the
menu's lists and holds the sections that belong in it.

```text
FUNCTION load(price_section)
  FOR EACH (section, price_list) IN config[price_section]
    entry = items[section]                 # inserted in name order
    entry.slot_index = 255                 # unassigned
    parse up to 5 comma-separated integers into entry.cost
    FAIL IF none parsed

    # A short list is extended by REPEATING THE LAST PRICE: an item
    # priced "500" costs 500 at every rank, and one priced "500,400"
    # costs 400 at ranks 1 through 4.
    WHILE fewer than 5 were parsed
      entry.cost[n] = entry.cost[n-1]

  FOR EACH list IN the menu's lists, in the menu's own order
    FOR EACH section IN config["buy_menu_items_place"][list's name]
      entry = items[section]
      FAIL IF not present               # priced items only
      FAIL IF entry.slot_index is already assigned
      entry.slot_index = the list's index
```

**Invariants** — The **repeat-the-last-price** rule is the decision. It lets the shipped
configuration write one number for items whose price does not vary by rank, and it means the
five-entry array is always full — no consumer ever has to handle a missing rank.

The placement pass iterates the menu's lists **in the menu's own enumeration order**, so the
slot index it assigns is the list's position, not a name. That couples the catalogue to the
menu's list enumeration; a rebuild that reorders the lists must reorder nothing else, because
the coupling is by position and is re-derived on every load.

Both failure modes are hard: an unpriced item in a placement list, and an item placed twice.
Either would leave the menu with an item it cannot price or cannot find.

## `GetItemCost` / `GetItemSlotIdx`

**Contract** — Lookups by section. Both assume the section exists; a miss is a programming
error, because everything that asks has already resolved the section through the catalogue.

## `GetItemIdx`

**Contract** — An item's position in the name order, or an out-of-band sentinel when the
section is unknown. **This one tolerates a miss** — unlike the two above — because it is what
resolves an item named by a mod or by a script, which may not be in the catalogue at all.

## `GetItemsCount` / `GetItemName`

**Contract** — The table as a sequence: how many, and the name at a position. The inverse of
`GetItemIdx`, and asserts the index is in range.

## `Dump`

**Contract** — Print the whole table in non-shipping builds and, in the same pass, assert that
**every** priced item received a slot. That assertion is the only completeness check on the
configuration; it is a validation pass wearing a diagnostic's clothing, exactly as in
[`Restrictions.cpp`](Restrictions.cpp.md), and a rebuild should keep the check while dropping
the printing.
