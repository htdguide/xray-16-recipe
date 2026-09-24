# src/xrUICore/TabControl/UITabControl.cpp

> Owns a group of tabs, guarantees exactly one is active, and turns tab selection into a saved setting and a pair of keyboard shortcuts.

**Needs** — [`UITabControl.h`](UITabControl.h.md) · [`UITabButton.h`](UITabButton.h.md) · [`Buttons/UIButton.h`](../Buttons/UIButton.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Data: User settings](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UITabControl.h`](UITabControl.h.md)
**Tier floor** — T3: a list, a current-name field and a key-binding lookup.

## Purpose

The tab strip along the top of the inventory, the options screen and the personal data
assistant. The control holds the tabs and the *name* of the active one; it does not hold the
pages. Whoever owns the control listens for the tab-changed message and shows or hides its own
panels. That split is the reason a tab control can be a few hundred lines: switching a tab is
purely an announcement.

Because tab selection is worth remembering between sessions, the control is also an options
item, storing the active tab's name as a setting.

## State

```text
RECORD TabControl EXTENDS Window, OptionsItem
  tabs              : list<TabButton>
  active_id         : text        # the current tab's name
  prev_active_id    : text        # the previous one, for the change notification
  backup_id         : text        # the options protocol's saved value
  idle_text_colour   : colour     # applied to a tab's label when it is built here
  idle_button_colour : colour
  active_text_colour   : colour
  active_button_colour : colour
  own_accelerators    : bool      # false: next/previous-tab keys
  tab_accelerators    : bool      # true:  each tab's own shortcut key
```

**Invariants**

- Every tab's name is non-empty and the control asserts it on insertion; lookup is by name.
- `active_id` names a tab that is in the list, except immediately after a reset, when both
  name fields are empty and no tab is depressed.
- `prev_active_id` trails `active_id` by exactly one change; the two are equal when settled.
  A change to a name that already equals `active_id` is dropped before anything is announced,
  which makes activation idempotent and stops an announcement echoing back into a loop.

## `OnTabChange` — the change protocol

**Contract** — the single funnel every tab change passes through, whether the player pressed
a tab, a shortcut fired, or a caller set the tab programmatically. It tells the outgoing tab
and the incoming tab about the new selection, then tells the control's owner.

```text
FUNCTION on_tab_change(new_id, old_id)
  outgoing <- tab named old_id
  incoming <- tab named new_id
  IF outgoing EXISTS THEN outgoing.receive(TAB_CHANGED naming incoming)
  IF incoming EXISTS THEN incoming.receive(TAB_CHANGED naming incoming)
  message_target.send(self, TAB_CHANGED)
```

**Notes** — both tabs are handed the *same* announcement, naming the incoming tab; each then
applies the group rule to itself, releasing or depressing. Only two tabs are touched, not the
whole list, which is what keeps a wide strip cheap — and which relies on the invariant that no
third tab was depressed.

The owner's notification carries no payload: the owner reads the active name back from the
control. That keeps the message vocabulary small and is the pattern throughout the toolkit.

## `SetActiveTab` / `SetActiveTabByIndex` / `SetNextActiveTab` / `ResetTab`

**Contract** — the ways the selection moves. Setting by name is the primitive: it drops a
no-op change, then advances the pair of names through the change protocol. Setting by index
resolves the index first and drops the change if it names the current tab. Stepping moves one
tab in a direction, optionally wrapping, and reports whether it moved — which is how a caller
can let the key fall through at the ends when wrapping is off. Resetting releases every tab
and blanks both names, leaving the control with no active tab at all.

**Notes** — stepping is the only place a *missing* selection is handled gracefully: with no
active tab the index is minus one, so stepping forward selects the first tab and stepping
backward wraps to the last.

## `OnKeyboardAction`

**Contract** — on a key press, two independent shortcut layers, each separately switchable.
The control's own layer maps the bound previous-tab and next-tab actions to a wrapping step.
The tabs' layer asks each tab whether the key is one of its accelerators and activates the
first that says yes. A key matched by either layer is consumed; everything else is refused,
including keys the children might have wanted — this handler does not walk the tree.

**Notes** — the two layers exist because they answer different questions: "cycle through the
tabs" is a property of the strip and is bound once in the key bindings, while "jump to the
inventory tab" is a property of that tab and is authored in the layout. Both default states
matter: the tabs' own shortcuts are on and the strip's cycling is off, so a screen must opt
in to cycling.

## `OnControllerAction`

**Contract** — the gamepad twin of the accelerator layer: a pressed axis or button is matched
against each tab's accelerators exactly as a key is. The strip's own cycling layer has no
gamepad equivalent — a deliberate gap, since the shoulder buttons that would cycle are already
spoken for.

## `SendMessage`

**Contract** — three cases. A tab-changed message from a tab in the list becomes a selection
change, dropped if the tab is already active. A focus arrival or departure on a tab is
re-raised to the control's owner *carrying the tab as payload*, so a screen can preview the
page under the highlight without selecting it. Anything else goes to the base.

**Notes** — the focus forwarding is what makes a gamepad-driven tab strip usable: the
highlight moves without committing, and the owner decides whether to preview. Note the payload
convention differs from the tab-changed message, which carries nothing — an inconsistency to
resolve in a rebuild, not to preserve.

## `AddItem`

**Contract** — two forms. The building form creates a tab from a name, a texture and a
rectangle, applies the control's idle colours, and adds it. The adopting form takes a
ready-made tab, shows it, enables it, puts it in latching mode, attaches it to the tree and
appends it to the list, asserting that it has a name. Both transfer ownership to the tree.

**Notes** — the idle colours are applied *at insertion* and never re-applied, so changing them
later restyles nothing. The active colours are declared here but are never applied by this
file at all — the XML reader and the owning screen use them. Recorded as an incomplete idea:
the type declares a four-colour scheme and implements half of it.

## `RemoveItemById` / `RemoveItemByIndex` / `RemoveAll`

**Contract** — removal detaches the tab from the tree and drops it from the list. Removal by
name asserts the name exists. Removal by index swaps the doomed tab with the last and pops,
which is cheap but **reorders the strip** — and since the strip's order is its visual order and
the stepping shortcut walks the list, a removal by index rearranges the tabs the player sees.
Removal by name preserves order.

**Notes** — the asymmetry is real and is a trap. A rebuild should make both order-preserving;
the cost is trivial for a list of half a dozen tabs, and the original's reason — pointer-sized
elements make the swap fast — does not apply at this scale.

Removing tabs does not touch the active name, so removing the active tab leaves the control
naming a tab that no longer exists. Nothing repairs it.

## The options-item protocol

**Contract** — the control remembers which tab was open. Reading pulls a name from the setting
and activates it, **falling back to the first tab** with a diagnostic when the name is unknown.
Saving writes the active name. Backing up and undoing copy the name; the change test compares
names.

**Notes** — the fallback is what makes a settings file survive a build that renamed or removed
a tab, and it is why the setting stores a name rather than an index: an index would silently
select the wrong page after a reorder, where a name fails loudly and recovers.

## `GetButtonById` / `GetButtonByIndex` / `GetActiveIndex` / `GetActiveId` / `GetPrevActiveId` / `GetTabsCount` / `GetButtonsVector`

**Contract** — lookup and inspection. The by-name search relies on equality being defined
between a tab and a name. An out-of-range index is refused rather than trusted; a name that
matches nothing answers "none"; the active index is minus one when nothing is active.

## `Enable`

**Contract** — enables or disables every tab and then the control itself. Note it does *not*
clear the selection — the source shows that behaviour removed — so disabling and re-enabling a
strip keeps the page the player was on.
