# src/xrGame/ui/UIActorMenu_action.cpp

> Every gesture the inventory screen understands — drop, double click, right click, focus,
> keys — routed by the *role* of the list it happened in.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`../../xrUICore/PropertiesBox/UIPropertiesBox.h`](../../xrUICore/PropertiesBox/UIPropertiesBox.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: event dispatch

## Purpose

One handler set is installed on every list
([`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md)), so each handler must work out
for itself what the gesture meant. It does that by resolving the source and destination lists
to **roles** and branching on those. The result is that adding a list to a screen requires no
new handler — only a role.

## `AllowItemDrops`

**Contract** — Consult the drop-permission table: is a drop from role *A* into role *B*
listed? One membership test. Every drop goes through it first.

## `OnItemDrop` — the drop handler

**Contract** — Called when a drag is released. Returns whether the list framework should
consider the drop handled and skip its own default move.

```text
FUNCTION on_item_drop(cell) -> handled
  clear the floating info panel
  source = cell.owner_list
  target = the list the drag is currently over
  IF either is missing THEN RETURN false        # dropped on nothing

  t_old = role(source); t_new = role(target)

  IF NOT allowed(t_old -> t_new)
    log the refusal
    RETURN true                                  # handled: consumed, nothing moves

  IF source = target
    # A drop inside the same list is not a move -- it is either a
    # re-placement or an item-onto-item gesture. Let script see it and
    # then let the list do its own default re-seat.
    notify script of the item-onto-item drop
    RETURN false

  CASE t_new
    trash        -> refuse quest items; a quick-slot drag merely unbinds the
                    slot; otherwise emit the drop event and clear the selection
    actor slot   -> resolve which slot this list means for this item, then place
    actor bag    -> place at the cursor
    actor belt   -> place at the cursor
    actor offer  -> stage for trade at the cursor
    partner offer     -> only from the partner's bag; stage
    partner bag       -> only from the partner's offer; un-stage
    loot bag     -> store at the cursor
    quick slot   -> bind

  offer the drop to the attach handler          # dropping a scope onto a gun
  IF script vetoes the item-onto-item drop THEN RETURN false
  re-derive placement-dependent state
  RETURN true
```

**Invariants**

- The two partner cases re-check the source role even though the table already did. That is
  not redundancy: the table permits partner-offer↔partner-bag in both directions, and these
  arms enforce that *only* those two roles participate, so a future table entry cannot
  accidentally let an actor's item into a partner's bag.
- A refused drop returns *handled*, so the list framework does not fall back to its default
  move. A same-list drop returns *not handled*, so it does.
- Dropping a quick-slot binding on the trash removes the binding without touching the item.
  The quick slot holds a reference, not the object.

## `DropItemOnAnotherItem` — the script hook

**Contract** — Give a script the chance to interpret one item being dropped onto another.
Resolves *which* item was landed on — the sole occupant if the destination holds exactly one,
otherwise whatever occupies the grid cell under the drop point — and calls a named script
function with both items and both roles.

**Invariants** — When nothing was landed on, the hook **is not called** and the answer is
"proceed". The hook is specifically for item-onto-item; plain moves must not reach it, or
every drag in the game would pay a script call.

A script returning false vetoes the rest of the drop.

## `OnItemDbClick` — the "put it where it belongs" gesture

**Contract** — Double-clicking an item moves it to the place the mode says is its natural
opposite. The rule is entirely about the *source* role:

```text
FUNCTION on_item_double_click(cell)
  select it; clear the floating info panel
  CASE role(cell.owner_list)
    actor slot     -> loot mode ? store in the loot bag : unequip to the bag
    actor bag      -> see below
    actor belt     -> to the bag
    actor offer    -> back to the bag (un-stage)
    partner bag    -> stage for purchase
    partner offer  -> un-stage
    loot bag       -> take to the bag
    quick slot     -> re-bind
  re-derive placement-dependent state
```

The bag case is the interesting one, because "the natural opposite" depends on the mode and on
the item:

```text
  # bag, in trade mode -> stage it.        bag, in loot mode -> store it.
  # bag, otherwise:
  IF not upgrade mode AND the item can be used THEN use it; done
  IF the item is a grenade or similar that is ACTIVATED rather than
     equipped THEN activate its slot; done

  IF the item's natural slot is non-persistent AND the item is
     already occupying that slot
    -> unequip it to the bag        # a second double-click takes it off
  ELSE IF it will not go into its natural slot
    -> try the belt; if that fails, force it into the slot, evicting
```

**Invariants** — The "already in its slot" test is what makes double-click a *toggle* rather
than a one-way equip. The fallback chain — slot, then belt, then forced slot — is the
decision: the player's intent is "equip this", and the forced variant is tried last because it
displaces something.

## `OnItemSelected` / `OnItemRButtonClick`

**Contract** — Selection records the cell and refreshes the info panel; right click does the
same and additionally opens the context menu. Both clear the "the floating info panel is
wanted" latch, so a deliberate click dismisses a tooltip that was about to appear.

## `OnItemFocusReceive` / `OnItemFocusLost` / `OnItemFocusedUpdate`

**Contract** — The three halves of hovering. Receiving focus marks the cell selected, applies
the highlight rules, sets the latch, and notifies script. Losing focus unmarks, clears every
highlight, and notifies script.

`OnItemFocusedUpdate` runs every frame the cursor dwells and is where the **delayed tooltip**
lives:

```text
FUNCTION on_item_focused_update(cell)
  mark the cell selected
  IF the highlights were cleared by something else THEN re-apply them

  # The dwell threshold is scaled by the game's time factor, so a
  # tooltip takes the same amount of PLAYER time when the simulation
  # is dilated for the inventory screen.
  IF now < cell.focus_received_time + tooltip_delay * time_factor
    RETURN

  IF a drag is in flight OR the context menu is open OR the latch is clear
    RETURN

  show the floating info panel for this cell
```

**Invariants** — Three suppressions, each for a different reason: during a drag the tooltip
would follow the cursor and obscure the drop target; with the context menu open the tooltip
would cover the menu; and the latch means a click has explicitly dismissed it.

Scaling the delay by the time factor is the load-bearing detail. The inventory screen dilates
time rather than pausing (see the chapter opener), so an unscaled delay would feel several
times longer with the inventory open than in the main menu.

## `OnKeyboardAction`

**Contract** — After the widget tree has had its chance, three bindings:

- **drop** — drops the selected item, if it is not a quest item and the actor owns it;
- **sprint** — repurposed inside this screen as the bulk-action key, with the modifier
  deciding direction: plain means take-all, held modifier means store-all. Dispatched by mode
  in `OnPressUserKey`: loot mode takes or stores everything, upgrade mode applies the described
  upgrade, trade and inventory do nothing;
- **quit, inventory, or the UI back action** — closes the screen.

All three consume the key whether or not they did anything, so the world never sees them.

**Notes** — Rebinding sprint inside a screen is not elegant and is the original's doing; what
matters for a rebuild is that the bulk actions need *some* key and that the modifier is what
distinguishes take from store.

## `OnMouseAction`

**Contract** — Forwards to the widget tree and then **always reports the event consumed**, so
that a click on the screen's background never reaches the world. An inventory screen with
click-through would fire the player's weapon.

## `OnDragItemOnTrash`

**Contract** — Fires when a drag enters or leaves the trash target. Entering with a
non-quest item installs the trash badge on the floating widget; leaving, or entering with a
quest item, removes it. That is the entire feedback for "this drop will destroy the item", and
it is why the floating widget has an overdraw hook at all.

The badge's size and texture are hardcoded and flagged as such in the source; a rebuild should
author them.

## `OnMesBoxYes` / `OnMesBoxNo`

**Contract** — The confirmation dialog's two answers, dispatched by mode. Only upgrade mode
acts: yes either performs a repair (when repair mode is latched) or forwards to the upgrade
panel; no clears the repair latch. Both then re-derive placement-dependent state, because
either may have changed an item's condition.
