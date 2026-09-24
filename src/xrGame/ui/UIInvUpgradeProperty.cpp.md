# src/xrGame/ui/UIInvUpgradeProperty.cpp

> The property list under an upgrade's description: one row per named property the game defines, each row shown only if some upgrade in the set carries it, and its displayed value computed by a script function given the list of contributing upgrades.

**Needs** — [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md) · [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`inventory_upgrade_property.h`](../inventory_upgrade_property.h.md) · [`inventory_upgrade_manager.h`](../inventory_upgrade_manager.h.md) · [`inventory_item.h`](../inventory_item.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md)
**Tier floor** — T3.

## Purpose

An upgrade changes numbers — accuracy, weight, recoil — and the player wants to see which.
This file renders that, and the shape it chose is worth stating because it is not obvious:
**the row set is built once from configuration, and each row decides for itself whether it
applies.**

That inverts the natural design, in which you would gather an upgrade's properties and build
a row per property. Building every row up front means the rows never move relative to each
other and the widgets are never destroyed, so a hover recomposes by toggling visibility
rather than by rebuilding a list.

## State

```text
RECORD PropertyRow EXTENDS Window
  property_id : text        # names an entry in the game's property registry
  icon        : Picture     # the property's own icon and tint, from the registry
  value       : Label
  text        : text        # the last composed value string

RECORD PropertyList EXTENDS Window
  rows      : list<PropertyRow>    # one per configured property, built once
  separator : optional<Picture>    # drawn above the rows
```

**Invariants**

- The row set comes from one configuration section whose keys are property identifiers. A key
  that does not resolve in the property registry is reported and its row dropped, so a bad
  configuration costs one row, not the screen.
- Every row's authored position is the same — the layout defines *a* row, and the list stacks
  the visible ones at run time.
- The list's height is `separator + the visible rows + 10`. An upgrade with no properties
  collapses the list to nothing, and an unknown upgrade collapses it to zero on both axes.

## Deciding whether a row applies

**Contract** — given a set of upgrades, a row collects the *sections* of those upgrades that
declare its property, joins them with commas, and asks the property's script function to turn
that list into a display string.

```text
FUNCTION compute(upgrades) -> bool
  contributors := empty
  FOR EACH upgrade IN upgrades
    FOR EACH property slot ON that upgrade        # a fixed small number of slots
      IF the slot names this row's property
        THEN append the upgrade's configuration section to contributors
  IF contributors is empty THEN RETURN false      # row does not apply
  RETURN the property's script function turned contributors into text
```

**Notes** — **the script function receives section names, not numbers.** It is expected to
read the numbers out of those sections itself and combine them however the property's
semantics demand — a weight delta sums, a multiplier multiplies, a best-of takes the maximum.
The engine cannot know which, so it hands over the list and takes back a string. That is the
seam's whole purpose here.

A function that declines to produce text blanks the row and reports that the row does not
apply, so a property whose script cannot make sense of the contributors is silently omitted
rather than shown empty.

An upgrade may declare the same property in more than one slot; the loop does not deduplicate,
so a section can appear twice in the list handed to the script. The shipped data does not do
this.

## Two subjects, one list

**Contract** — the list is filled either from a **single upgrade** — what the description
panel shows on hover — or from **an item's installed upgrade set** — what the bench shows for
the item as a whole. Both go through the same computation; the first wraps its one upgrade in
a list of one.

**Notes** — that is the reason the computation takes a *set* rather than an upgrade. The same
row, the same script function, and the same arithmetic serve "what would this upgrade do" and
"what have all my upgrades done", which is how the two readings stay consistent.

An unknown upgrade collapses the list to zero size without computing anything.

## Helpers

**Contract** — reading a numeric parameter out of a configuration section, tolerating a
missing section, a missing key and an empty value, and reporting whether it found one.

**Notes** — declared on the row and unused by it; it is there for script-side value
computation that was later moved entirely into Lua. A rebuild drops it.
