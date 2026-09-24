# src/xrGame/ui/UIActorMenu_script.cpp

> Repair and upgrade decisions are not the engine's to make: it asks a script, and the script
> also writes the question the player is shown.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`../../xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: script calls and a binding table

## Purpose

Two things: the repair and upgrade *policy*, which is delegated wholesale to named script
functions, and the frozen script surface of the inventory screen and the personal terminal.

The delegation is the decision worth carrying. Whether a mechanic will repair a given item,
what it costs, and what the confirmation dialog says are all authored per game in Lua. The
engine contributes only the item, its condition and the mechanic's profile name.

## `TryRepairItem`

**Contract** — Wired to the repair button and to the context-menu repair entry. Asks script
twice and then opens the appropriate dialog.

```text
FUNCTION try_repair_item()
  item = the item on the bench
  IF none THEN RETURN
  IF item.condition > 0.99 THEN RETURN            # nothing to repair

  # A consumable is repairable only if its own configuration opts in.
  IF item is consumable AND NOT item.config.allow_repair THEN RETURN

  mechanic = partner.profile_name

  can   = script("inventory_upgrades.can_repair_item")(item.section, item.condition, mechanic)
  text  = script("inventory_upgrades.question_repair_item")(item.section, item.condition,
                                                            can, mechanic)
  IF can
    latch repair mode
    show a yes/no dialog with text
  ELSE
    show an acknowledge-only dialog with text
```

**Invariants** — Both script functions are **required**; their absence is fatal and names the
item. That is deliberate: silently skipping repair would look like a broken mechanic.

The *same* script function writes both the offer and the refusal — it is told whether the
repair is possible and returns the matching sentence — so the engine never has to choose
between two strings.

The 0.99 threshold matches the one the context menu uses to decide whether to offer repair at
all; keeping them equal is what stops the menu offering something the button then declines.

## `RepairEffect_CurItem`

**Contract** — Runs on a yes. Calls an *optional* script effect with the item and its
pre-repair condition (for a sound, a message, a money deduction), then **sets the condition to
full**, separates the item from its stack, and refreshes its condition bar.

**Invariants** — The effect is optional and the repair is not: the engine restores condition
whether or not a script watched. Separating from the stack must happen after the condition
changes, so that the re-grouping sees the new value — see
[`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md).

## `CanUpgradeItem`

**Contract** — Ask script whether a mechanic will work on an item at all, given the item's
section and the mechanic's profile name. Required; absence is fatal.

## `CurModeToScript`

**Contract** — Tell script the screen's mode changed, by number. Optional — the call is
skipped when the function is absent — because not every game's scripts care.

## `script_register` — the frozen surface

**Contract** — Exports three things, and the names are frozen by conformance criterion 10.

The list-role enumeration, under the name `EDDListType`, with all ten values:
`iActorBag`, `iActorBelt`, `iActorSlot`, `iActorTrade`, `iDeadBodyBag`, `iInvalid`,
`iPartnerTrade`, `iPartnerTradeBag`, `iQuickSlot`, `iTrashSlot`.

The inventory screen, under the name `CUIActorMenu`, deriving from the dialog base, with:
`get_drag_item`, `highlight_section_in_slot`, `highlight_for_each_in_slot`,
`refresh_current_cell_item`, `IsShown`, `ShowDialog`, `HideDialog`, `ToSlot`, `ToBelt`.

The personal terminal, under the name `CUIPdaWnd`, with: `IsShown`, `ShowDialog`,
`HideDialog`, `SetActiveSubdialog`, `SetActiveDialog`, `GetActiveDialog`, `GetActiveSection`,
`GetTabControl`.

And a free namespace `ActorMenu` with four accessors: `get_pda_menu`, `get_actor_menu`,
`get_menu_mode`, `get_maingame`.

**Notes** — The terminal's surface is exported from the inventory screen's file rather than
its own. That is arbitrary and a rebuild may move it; what may not move is any of the names.
