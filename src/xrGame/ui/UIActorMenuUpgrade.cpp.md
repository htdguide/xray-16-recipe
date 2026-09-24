# src/xrGame/ui/UIActorMenuUpgrade.cpp

> Upgrade mode: choosing which item is on the mechanic's bench, and splitting it out of its
> stack so the upgrade lands on one object and not on five.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UITalkWnd.h`](UITalkWnd.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: list orchestration

## Purpose

The upgrade bench is the simplest of the four modes: one bag, one selected item, and a panel
of upgrade options supplied by [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md).
What this file owns is the *selection* — including the one operation that has no analogue in
the other modes, separating the selected item from its stack.

Upgrade mode exists only in the universal layout dialect; in the split dialect there is no
upgrade window and every operation here degrades to nothing.

## `InitUpgradeMode` / `DeInitUpgradeMode`

**Contract** — Entry shows the partner panel, the upgrade panel and the quick slots, hides the
partner's money (an upgrade is priced per option, not as a balance), fills the inventory bag,
and tells the partner it is trading — a mechanic is a trading character.

Exit hides the panel, clears its current upgrade, disables the repair button, **unmarks the
selected cell**, stops the partner trading, and — if the talk screen is underneath — tells it
to re-evaluate its question list, because upgrading can change what there is to talk about.

**Invariants** — The mark must be cleared on exit, because the cell outlives the mode and a
mark left behind would show in the inventory screen.

## `SetupUpgradeItem`

**Contract** — Point the bench at whatever cell is currently selected. Unmarks the previously
marked cell, marks the new one, asks script whether the item can be upgraded at all, and hands
both to the upgrade panel. Hides the upgrade-detail popup, since the selection changed.

**Notes** — The *mark* is a distinct highlight channel from selection (see
[`UICellItem.h`](UICellItem.h.md)). Selection is transient and follows the cursor; the mark
says "this is the item on the bench" and must survive the cursor moving away.

## `TrySetCurUpgrade`

**Contract** — Apply whichever upgrade the detail popup is currently describing, as if it had
been double-clicked. Bound to a keyboard shortcut so the bench is usable without a mouse. Does
nothing when no upgrade is being described.

## `SetInfoCurUpgrade`

**Contract** — Show the detail popup for one upgrade of one item, and nudge it so it stays
inside the canvas rather than running off the edge beside the screen. Returns whether the
popup had anything to show; a null upgrade always answers no, which is how the popup is
dismissed.

## `SeparateUpgradeItem`

**Contract** — The one operation unique to this mode. An upgrade applies to a single object,
but the inventory shows identical objects merged into a stack — so before an upgrade is
applied the selected item is taken out of its list and put straight back, which makes the list
re-run its auto-grouping and re-seat it as its own cell.

```text
FUNCTION separate_upgrade_item()
  IF nothing is marked THEN RETURN
  owner = the marked cell's list
  IF the marked cell is not in the actor's bag THEN RETURN   # slots cannot be worked on

  unmark it
  cell = owner.remove(marked, keep as root)
  owner.add(cell)                       # re-inserted; grouping decides where it lands
```

**Invariants** — The remove/add round trip is not a no-op. Removing with "keep as root" takes
the whole stack head out; re-adding runs it back through the list's merge test, and the item
whose condition or upgrade set has just changed no longer matches its former stack-mates. A
rebuild that skips this step will apply an upgrade to a stack and show it on all five.

The mode restriction — bag only — is because a slot list has no grouping and nothing to
separate from.

## `UpdateUpgradeItem`

**Contract** — Called every frame in upgrade mode; currently does nothing. Kept as the hook
the per-frame update calls, which a rebuild may simply not have.

## `get_upgrade_item`

**Contract** — The item on the bench, or nothing.
