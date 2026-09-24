# src/xrGame/ui/UIActorMenuInitialize.cpp

> Building the inventory screen from shipped layout documents — in either of two incompatible
> dialects — and writing down, once, which kind of list may receive a drop from which.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`UIOutfitSlot.h`](UIOutfitSlot.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIOutfitInfo.h`](UIOutfitInfo.h.md) · [`UIActorStateInfo.h`](UIActorStateInfo.h.md) · [`UITradeBar.h`](UITradeBar.h.md) · [`UIWeightBar.h`](UIWeightBar.h.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [`../../xrUICore/PropertiesBox/UIPropertiesBox.h`](../../xrUICore/PropertiesBox/UIPropertiesBox.h.md) · [`../../xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: widget construction from data

## Purpose

Everything the inventory screen is made of, and two decisions that are stated nowhere else:
**which layout dialect the data is in**, and **which drops are legal**. Both are tables, and
both are the kind of thing a rebuild gets wrong by inventing its own.

## The two layout dialects

The screen exists in two incompatible authorings, selected at construction by which game's
data is loaded:

- **The universal dialect** — one document, `actor_menu.xml`, holding all four modes'
  widgets. Modes show and hide subsets of one shared tree. Several logical lists alias the
  same widget: the loot-side actor bag *is* the inventory bag.
- **The split dialect** — three documents, `inventory_new.xml`, `trade.xml` and
  `carbody_new.xml`, one per mode, each holding a complete and separate tree. Upgrade mode has
  no document at all and is unreachable.

**Invariants** — In the split dialect the screen's own rectangle is computed as the bounding
box of the three sub-windows, because event routing descends through the parent's rectangle
and a screen smaller than its children would swallow no events for them. This is the one place
the toolkit's "a child is neither clipped nor resized by its parent" rule (chapter 15) has to
be worked around rather than relied on.

Because of the dialects, **nearly every widget pointer in the screen may be absent**, and
every use is guarded. A rebuild that assumes a complete widget set will crash on one of the
three shipped games.

## The optional-element convention

Construction uses one helper family throughout: *create widget of kind K from element E of
document D, attach to parent P, and fail loudly if E is missing — unless told the element is
optional, in which case return nothing*.

That flag is the entire error policy of the chapter:

- **Required and missing** → hard failure naming the element. The screen cannot work without
  it, and a silent half-built screen is worse than a stop.
- **Optional and missing** → the widget is simply absent, and every later use checks.

Element names are therefore **frozen by the shipped data**: the screen finds its widgets by
name at construction, and a renamed element is a missing element.

## The list table

The universal dialect builds its sixteen lists from one literal table, and the table is the
decision:

```text
# id                     element                    condition bar          highlighter                blocker           required
  inventory knife         dragdrop_knife             progess_bar_knife      inv_slot1_highlight        -                 no
  inventory pistol        dragdrop_pistol            progess_bar_weapon1    inv_slot2_highlight        -                 yes
  inventory automatic     dragdrop_automatic         progess_bar_weapon2    inv_slot3_highlight        -                 yes
  inventory outfit        dragdrop_outfit            progess_bar_outfit     outfit_slot_highlight      -                 yes
  inventory helmet        dragdrop_helmet            progess_bar_helmet     helmet_slot_highlight      helmet_over       no
  inventory belt          dragdrop_belt              -                      artefact_slot_highlight    belt_list_over    yes
  inventory detector      dragdrop_detector          -                      detector_slot_highlight    -                 yes
  inventory bag           dragdrop_bag               -                      -                          -                 yes
  trade actor             dragdrop_actor_trade       -                      -                          -                 yes
  trade actor bag         dragdrop_actor_trade_bag   -                      -                          -                 yes
  trade partner           dragdrop_partner_trade     -                      -                          -                 yes
  trade partner bag       dragdrop_partner_bag       -                      -                          -                 yes
  loot bag                dragdrop_deadbody_bag      -                      -                          -                 yes
  loot actor bag          (aliases the inventory bag)                                                                    -
  trash                   dragdrop_trash             -                      -                          -                 no
  inventory backpack      dragdrop_backpack          -                      backpack_slot_highlight    -                 no
```

Three things travel with a list besides the list itself:

- a **condition bar** drawn beside the slot, showing the wear of whatever occupies it;
- a **highlighter** — a static drawn over the slot when the highlight rules
  ([`UIActorMenu.cpp`](UIActorMenu.cpp.md)) say this slot would accept the item under the
  cursor — positioned by an offset the element itself carries;
- a **blocker** — a static drawn over the slot when it is unavailable, which is how a helmet
  slot is struck out under a sealed outfit.

**Notes** — Two element names are misspelled in the shipped data (`progess_bar_…`) and the
code spells them the same way. They are frozen: the data cannot be corrected without breaking
every existing mod.

The knife, helmet, detector, backpack, trash and quick-slot lists are optional because
different games in the series have different slot sets. The screen's rules are written so that
an absent slot is simply unreachable, never an error.

## The quick slots

Built separately as a *reference* list — a fixed row of slots holding item **sections** rather
than item instances (see [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md)). It is
initialised with two name templates, one for the slot's caption element and one for the
localization key of its key binding, both taking the slot index. That is how four slots are
labelled with four different key names from one element.

## `InitAllowedDrops` — the drop-permission table

**Contract** — Fills, for each destination role, the set of source roles it accepts. Every
drop in the screen is checked against this table first; a drop not listed is refused with a log
line and nothing else happens.

```text
into trash        <- bag, slot, belt, quick slot
into slot         <- bag, slot, actor's trade side, loot bag
into bag          <- slot, belt, actor's trade side, loot bag, bag, quick slot
into belt         <- bag, actor's trade side, loot bag, belt
into actor trade  <- slot, bag, belt, actor trade, quick slot
into partner bag  <- partner trade side, partner bag
into partner trade<- partner bag, partner trade
into loot bag     <- slot, bag, belt, loot bag
into quick slot   <- bag, actor's trade side, quick slot
```

**Invariants** — Read the table for what it *forbids*, which is where the game rules are:

- **The two trade sides are sealed from each other.** Nothing may move from the actor's side
  to the partner's or back by dragging; that is what the trade button is for. The partner's
  two lists exchange only with each other, and the actor's only with the actor's own storage.
- **Nothing may be dropped into a partner's bag**, so the player cannot give items away by
  dragging.
- **The trash accepts from anywhere the actor owns**, including the quick slots, and from
  nowhere else — you cannot destroy a partner's property.
- **A slot may receive from another slot**, which is how an item is moved between the two
  interchangeable weapon slots.
- Every self-to-self pair is present, because reordering within a list goes through the same
  check.

## `InitCallbacks` / `BindDragDropListEvents`

**Contract** — Wires the buttons to their handlers and gives **every** list the same eight
gesture callbacks: drop, start drag, double click, select, right click, focus received, focus
lost, focused update. Uniformity is the point — the handlers in
[`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) branch on the list's *role*, so every
list can safely share one handler set.

The trash list is the exception: it takes only the drop callback plus a **drag-event** callback
that fires when a drag enters or leaves it, which is what installs the trash badge on the
floating widget.

Registering a callback on an absent widget is a no-op, which is what makes the optional-widget
policy survive into the wiring.

## `InitSounds`

**Contract** — Loads the ten action sounds from an `action_sounds` element of the layout
document, saving and restoring the document's local root around the read. Sounds are named in
data, not in code.

## Construction order

**Contract** — The order is load-bearing and is:

1. Build the context menu, the two message boxes, the trade bars, the weight bars and the
   condition panel — all things that exist in both dialects and are attached later.
2. Load the layout document(s) and build the mode trees.
3. Load the sounds from the same document.
4. Wire the callbacks — after the widgets exist.
5. Fill the drop table — independent of widgets.
6. Attach and hide the context menu.
7. Clear the selection, the actor, the partner and the container.
8. **Run every mode's exit step.** This is what leaves the screen in a consistent hidden state:
   the entry steps show things, so without a matching exit pass the screen would open with
   several modes' widgets visible at once.
9. Remove the exit button from the navigation-focus registry, because it has a keyboard and a
   gamepad shortcut and would otherwise be a focus stop the player keeps hitting.

## `UpdateButtonsLayout`

**Contract** — In the universal dialect the exit button's horizontal position depends on
whether a trade or take-all button is showing beside it: it sits flush past the button when
one is visible, and half-overlapping when none is, so the pair reads as centred either way.
The split dialect authors the position and this does nothing. Also refreshes the quick slots'
key labels, since a key binding can change while the screen is closed.

## `ShowIfExist`

**Contract** — Show or hide a widget that may be absent, and report whether it was there. The
whole chapter's optional-widget idiom in one line; the original apologises for it in a comment
and it is nonetheless the right shape.
