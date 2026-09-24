# src/xrGame/ui/UIMpTradeWnd.cpp

> The buy screen's navigation and commit: browsing the category tree, the five loadout buttons,
> and the two ways the screen can end.

**Needs** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UITabButtonMP.h`](UITabButtonMP.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIDialogHolder.h`](../UIDialogHolder.h.md) · [`Actor.h`](../Actor.h.md) · [`game_cl_deathmatch.h`](../game_cl_deathmatch.h.md) · [`xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T3.

## Purpose

The screen's outer behaviour: where the player is in the store, which loadout button does what,
and what happens on accept or cancel. The item economics are in the sibling files.

## Browsing the store

**Contract** — the store is browsed by moving a cursor in the category tree and rebuilding the
shop panel from wherever the cursor now is.

```text
FUNCTION update_shop()
  shop_panel.detach_all()
  at_root <- store.cursor_is_root()
  back_button.enabled <- NOT at_root
  IF at_root THEN tab_strip.clear_selection()
  set_current_item(none)
  IF at_root THEN RETURN                     # the tab strip is the whole display at the root

  IF store.current_level has children THEN
    FOR EACH child: attach its tab button to the shop panel, tagged with the child's name
  ELSE
    attach the shop list to the shop panel, clear it, and for each item section at this
    leaf, place a shelf copy of it — see RenewShopItem in the trade twin
```

**Notes**

- **The shop panel is emptied and refilled on every move.** It holds either a row of category
  buttons or the shelf list, never both, and the buttons are *the tree's own* — each category
  node owns its button for the life of the screen, so attaching is free and no button is
  created here.
- Selecting a top-level tab resets the cursor to the root and descends by the tab's identity,
  rather than remembering where in that branch the player was. Descending into a sub-category
  moves down one level from wherever the cursor is. The back button moves up one.
- Every navigation action first destroys any item currently being dragged, because the widget
  it is dragging is about to be detached. That guard appears at the head of nearly every
  handler in the screen and is the whole of the screen's drag-safety story.

## The loadout buttons

**Contract** — five buttons apply a saved loadout; three more overwrite one with the current
selection. Applying a loadout with the modifier key held instead dumps it to the log, which is
a development affordance. `TryUsePreset` — the entry point the game layer uses, as opposed to
the button — additionally refuses when the loadout costs more than the player has; the buttons
themselves are disabled in that case by the money refresh.

**Notes** — the *reset* button applies a reserved loadout that was stored when the screen
opened, so "reset" and "apply the loadout I arrived with" are the same operation. That is why
there are more loadout slots than buttons: three player slots, the default, the last used, the
arrival state, and one scratch slot the money display re-stores every thirty frames.

## `Show`

**Contract** — on either transition, ask the actor to hide or show its weapon for the
buy-menu reason, which is what puts the player's hands down while the screen is up. On showing,
drop any pointer capture and clear the two message lines. On hiding, destroy any dragged item
and return every player-side item to the shelf.

**Notes** — the weapon-hide call names a *reason*; several screens can independently ask for the
weapon to be hidden and the actor holds the union. A rebuild that uses a boolean will find two
screens fighting over it.

## Accept and cancel

**Contract** — accept deletes the automatically-added convenience items, destroys any dragged
item, stores the current selection into the *last used* loadout, closes, and tells the
multiplayer game layer the buy menu was confirmed. Cancel destroys the dragged item, closes, and
tells the game layer it was cancelled.

**Notes** — **the screen does not send the purchase.** It ends by notifying the game layer,
which then reads the screen's lists through the shared buy-screen interface and builds the
network message. That indirection is what lets the same game code drive either buy screen, and
it is the chapter's "a UI action becomes a game action through an event" rule at its most
literal.

Note also that accept *removes* the convenience items before storing the loadout, so a loadout
never records ammunition the player did not explicitly choose.
