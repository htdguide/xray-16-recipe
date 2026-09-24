# src/xrGame/ui/UIActorMenu.cpp

> The mode machine behind the inventory screen, and the highlight rules that tell the player
> at a glance which slot an item fits and which ammunition fits the gun under the cursor.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIActorStateInfo.h`](UIActorStateInfo.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`UIActorMenu.h`](UIActorMenu.h.md); callers name that, not this file.
**Tier floor** — T2: widget orchestration and per-frame polling; no device-facing layout

## Purpose

One window is four screens. This file holds the part that makes that true: the transition
between modes, what each frame polls, how a widget is resolved to its *role*, and the
highlight rules — which are not decoration, they are how the player is told what the drop
rules in [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) will permit.

## State

See [`UIActorMenu.h`](UIActorMenu.h.md). The invariants that this file enforces:

- **The actor, the partner and the container may only be set while the screen is hidden.**
  Every setter asserts it. Changing who you are trading with mid-screen would leave the lists
  filled from a stranger's inventory.
- **A partner and a container are mutually exclusive.** Setting either clears the other. Loot
  mode reads whichever is present.
- `m_highlight_clear` is a latch, not a cache: it says "the highlight lists are known to be
  empty", and it exists so that a per-frame focus update can re-apply a highlight that a mode
  change wiped without re-applying it every frame.

## `SetMenuMode` — the mode machine

**Contract** — The only way the screen changes what it is. Idempotent for the same mode in the
sense that it skips the transition, but **not** a no-op: it always refreshes the actor's money
and weight and re-lays the buttons, which is why showing the screen calls it with the current
mode.

```text
FUNCTION set_menu_mode(mode)
  current item = none                  # clears the info panel and hides the context menu
  hint text = none
  transient message = none

  IF mode != current_mode
    run the exit step for current_mode          # DeInit*, per file
    hide the in-game minimap                    # the screen draws its own copy
    current_mode = mode
    run the entry step for mode                 # Init*, per file

    refresh the actor's character panel
    IF mode is neither undefined nor inventory
      refresh the partner's character panel
    re-apply the equipped highlight
    tell script the mode changed

  IF an actor is set
    refresh the outfit-derived belt capacity
    refresh money and weight
  re-lay the buttons
```

**Invariants** — Exit before entry, always. The two steps touch overlapping widgets — trade
mode hides the inventory bag list and shows the trade bag list, which in one layout dialect
are the *same widget* — and running them out of order leaves a list hidden that should be
shown.

The partner panel is refreshed only outside inventory mode because inventory mode has no
partner; refreshing it would clear a panel that another mode still owns in the shared-widget
dialect.

## `Show` / `ShowDialog`

**Contract** — Showing re-enters the current mode, plays the open sound, and refreshes the
actor's condition panel immediately so the screen is correct on its first frame rather than
after one update. Hiding plays the close sound and transitions to undefined, which runs the
current mode's exit step — that is the only place mode cleanup is guaranteed.

`ShowDialog` additionally, **when the current input device is a controller**, moves navigation
focus to the first item of whichever bag list the current mode uses. A controller has no
cursor to place, so the screen must hand it a starting point or the player cannot move.

## `NeedCenterCursor`

**Contract** — Whether the dialog framework should warp the cursor to the centre when the
screen opens. The answer is **yes only when the mode's bag list is empty**. A player opening
an inventory that has items wants the cursor where they left it; a player opening an empty one
has nothing to point at and the centre is the least surprising place.

## `StopAnyMove`

**Contract** — Whether the actor is frozen while this screen is up. **Inventory mode alone
answers no** — the player may keep walking with the inventory open, which is a deliberate
series behaviour. Trade, upgrade and loot all freeze the actor, because all three require
standing next to somebody.

## `Update` — what each frame polls

**Contract** — Called once per frame while shown. The common part refreshes the actor's
condition panel. Then, per mode:

```text
FUNCTION update()
  refresh the actor condition panel            # every mode

  CASE inventory
    update the clock readout to game time, minute resolution
    update the in-screen minimap
  CASE trade
    # The partner's inventory can change underneath us -- a script may
    # give or take. Rather than a change notification, the inventory
    # carries a modification counter and the screen watches it.
    IF partner.inventory.modify_counter != remembered
      refill the partner's bag list
    check the distance to the partner
    age the transient message and retire it when it expires
  CASE upgrade
    refresh the upgrade panel
    check the distance to the partner
  CASE loot
    # deliberately no distance check: containers may be looted at range
```

**Invariants** — The modification counter is the whole mechanism for "the partner's inventory
changed"; there is no event. A rebuild that uses events must still tolerate the case the
counter handles, which is a change made by a script between frames.

## `CheckDistance`

**Contract** — Trade and upgrade close themselves when the actor moves more than **3 metres**
from the partner or the container. One exception: a partner in the "awareness" mode some
characters have is exempt and holds the screen open at any range.

**Notes** — The 3 is the same number the game uses for "close enough to interact" elsewhere,
and it is what makes walking away a way to end a conversation. Loot mode was deliberately
exempted from the check so that a container's contents can be inspected while backing away.

## `GetListType` / `GetListByType`

**Contract** — The two directions of the widget↔role mapping the drop rules are written
against.

`GetListType` maps a widget to its role by identity comparison against every list the screen
owns, in a fixed order. A widget that is no list is a programming error.

`GetListByType` maps a role back to a widget and is **mode-dependent**, which is the
interesting direction:

```text
FUNCTION list_for_role(role) -> list
  CASE actor bag
    IF mode = trade THEN RETURN the trade bag list
    IF mode = loot  THEN RETURN the loot-side actor bag list
    RETURN the inventory bag list
  CASE loot bag    RETURN the loot bag list
  CASE actor belt  RETURN the belt list
  otherwise        FAIL              # only these three roles are addressable this way
```

**Invariants** — Only three roles are resolvable back to a widget, because only those three
are ever asked for by role rather than reached by gesture. The asymmetry is deliberate: a slot
is found by slot index, not by role, since there are many slots and one role.

In one layout dialect the loot-side actor bag *is* the inventory bag; in the other it is a
separate widget. Code must never assume they differ — several operations compare them to
decide whether to refill.

## `SetCurrentItem` — the selection and the info panel

**Contract** — Records which cell is selected and drives the mode-specific item-info panel
from it. Leaving repair mode, hiding the context menu and, in upgrade mode, re-targeting the
upgrade bench are all part of the same act.

The interesting part is the price:

```text
FUNCTION set_current_item(cell)
  repair_mode = false
  current = cell
  IF cell is none THEN clear the info panel
  hide the context menu

  info = the item-info panel for the current mode
  IF info exists
    price = unknown
    compare_item = none
    tip = none
    IF cell holds an item
      # Compare against whatever already occupies this item's natural
      # slot, so the panel can show a side-by-side.
      IF item.base_slot is a real slot
        compare_item = actor.inventory.item_in_slot(item.base_slot)

      IF mode != trade
        price = item.cost                       # the catalogue price
      ELSE
        # In trade the price depends on WHICH SIDE owns the item: the
        # partner buys at one rate and sells at another.
        selling = (item's current inventory owner is the actor)
        price = partner_trade.price(item, selling)

        # Ammunition is priced per stack, not per box.
        IF item is ammunition
          FOR EACH child IN cell.stack
            price = price + partner_trade.price(child.item, child is actor's)

        # Two refusals, each with its own explanatory tip:
        IF item is untradeable
           OR (the partner will not buy this section AND the actor owns it)
          price = unknown; tip = "this cannot be traded"
        ELSE IF item.condition < partner.minimum_buy_condition
          price = unknown; tip = "this is too worn to be traded"

    info.show(cell, compare_item, price, tip)

  IF mode = upgrade THEN re-target the upgrade bench
```

**Invariants** — The order of the two refusals matters: a quest item that is also worn must
report the untradeable reason, not the condition reason, because the condition one suggests
repair would help.

"Price unknown" and "price zero" are different states and are drawn differently; the sentinel
is an out-of-band value, and a rebuild needs an explicit absent price.

The partner's willingness to buy is checked **only for items the actor owns**. Items on the
partner's side are being sold *to* the actor, and the partner's buy list does not govern that.

## `InfoCurItem`

**Contract** — The same computation again, against the *other* item-info panel — the one that
belongs to the newer layout dialect and floats beside the cursor rather than sitting in the
frame. After filling it, it is nudged so that it stays inside the canvas rather than running
off the right edge.

**Notes** — This is duplicated logic, not a variant: the two panels compute the same price the
same way. The duplication is a defect of the original; a rebuild should compute the price once
and hand it to whichever panels exist.

## The highlight rules

Four independent highlight channels write into cells and lists, and this file owns all of
them. They exist to answer, without the player trying it, "where can this go, and what does
it go with".

### `highlight_item_slot`

**Contract** — Given the cell under the cursor, light up **the slot list(s) the item would go
into**. Suppressed entirely while a drag is in flight, because during a drag the drop target
is the thing being indicated and two indications would conflict.

The mapping is by item kind *and* the item's natural slot, both: a weapon whose natural slot
is one of the two weapon slots lights **both** weapon slots, because either will take it; a
helmet lights the helmet slot; an outfit the outfit slot; a detector the detector slot; a
backpack the backpack slot. Consumables light the quick slots and artefacts light the belt —
except when the item is already there, in which case nothing lights, since the highlight is
advice about a move.

### `highlight_armament`

**Contract** — Given the item under the cursor, mark every cell in a bag list that is
*related* to it. Three relations, all applied:

- ammunition that fits the weapon under the cursor — including the grenade launcher's separate
  ammunition list when one is attached;
- weapons that take the ammunition under the cursor — again including launcher ammunition;
- weapons that can take the attachment under the cursor, and attachments that fit the weapon
  under the cursor.

**Invariants** — "Fits" is decided by the *item's own* attach test, not by a table here, so
modded weapons and modded attachments participate for free. Ammunition matching is by interned
section name identity, not string comparison, which is why it is cheap enough to run over
every cell in a bag on every focus change.

Which bag lists are marked depends on the mode: inventory and upgrade mark one, trade marks
all four, loot marks both.

### `highlight_equipped`

**Contract** — Marks, in the actor's bag list, every item that is *currently equipped* —
meaning the item is in its own natural slot in the actor's inventory and is flagged as wanting
the marking. Re-applied on every mode change and every item movement.

**Notes** — This is the one highlight that is not about the cursor. It exists because the
shipped games let an equipped item also appear in the bag list, and without the mark the
player cannot tell which copy is the one in their hands.

### `clear_highlight_lists`

**Contract** — Turn all of it off, and set the latch. Called whenever anything moves, because
every rule above may have become stale.

## `HighlightSectionInSlot` / `HighlightForEachInSlot`

**Contract** — The script-facing highlight hooks: mark every cell in a named list whose item
is of a given section, or for which a script predicate returns true. Both fall back to the
inventory bag list when the named role does not resolve, and both clear the latch so the
marks survive the next frame.

**Notes** — The `slot_id` argument both take is accepted and ignored. That is dead surface
preserved because it is part of the exported signature.

## `ClearAllLists`

**Contract** — Empty every list, destroying the cells. Run before any refill and at
destruction. The list array is walked explicitly rather than in a loop because several entries
alias and must not be cleared twice.

## `ShowMessage` / `CallMessageBoxYesNo` / `CallMessageBoxOK`

**Contract** — The screen reports refusals two ways, and which one it uses depends on the
layout dialect. When the layout supplies a message box, the message is a modal. When it does
not — the older dialect — the message becomes a **transient overlay static** with a lifetime,
registered with the in-game UI and retired by the per-frame update.

**Invariants** — `CallMessageBoxYesNo` with **no message box available answers yes
immediately**. That is a deliberate degradation, not an oversight: the older dialect has no
confirmation dialog, and refusing to act would make repair and upgrade unreachable there.

## `ResetMode`

**Contract** — The transition into undefined: clear the lists, release any pointer capture,
hide the context menu, clear the selection. Releasing the capture is the load-bearing part — a
screen closed mid-drag must not leave the capture pointing at a destroyed cell.

## `CanSetItemToList`

**Contract** — Given an item and a slot list, may the item go there, and into which slot. The
natural answer is "into its own base slot, if that is this list". One game adds an exception:
the two weapon slots are **interchangeable**, so an item whose natural slot is the first may
be placed in the second and vice versa. That exception is gated on which game's data is
loaded, because the other games' slot layouts do not permit it.

## `UpdateActorMP`

**Contract** — In multiplayer the actor panel shows the local player's round money and a fixed
placeholder portrait rather than a character record, because multiplayer players have no
character record. Falls back to clearing the panel when there is no game or no local player.

## `GetCurrentItemAsGameObject`

**Contract** — The selected item as the script-visible facade, or nothing. Part of the frozen
script surface; see [`UIActorMenu_script.cpp`](UIActorMenu_script.cpp.md).

## `Draw`

**Contract** — Draws the in-game minimap and the main indicators *first*, beneath the screen,
so that health and the minimap remain visible with the inventory open; then the screen; then
the floating item-info panel, the hint and the transient message, each of which must be over
everything because each is a comment on something underneath.
