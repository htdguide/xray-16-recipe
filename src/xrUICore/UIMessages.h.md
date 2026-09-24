# src/xrUICore/UIMessages.h

> The single flat vocabulary of notifications every widget in the engine can send to its owner — and, because the same numbers are exported to Lua, a frozen part of the script surface.

**Needs** — _(none — a vocabulary, not a dependency)_ · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIButton.h`](Buttons/UIButton.h.md) · [`UICheckButton.cpp`](Buttons/UICheckButton.cpp.md) · [`UIRadioButton.cpp`](Buttons/UIRadioButton.cpp.md) · [`UIComboBox.cpp`](ComboBox/UIComboBox.cpp.md) · [`UICustomEdit.cpp`](EditBox/UICustomEdit.cpp.md) · [`UIListBox.cpp`](ListBox/UIListBox.cpp.md) · [`UIListBoxItem.cpp`](ListBox/UIListBoxItem.cpp.md) · [`UIListItemEx.cpp`](ListWnd/UIListItemEx.cpp.md) · [`UIListWnd.cpp`](ListWnd/UIListWnd.cpp.md) · [`UIMessageBox.cpp`](MessageBox/UIMessageBox.cpp.md) · [`UIPropertiesBox.cpp`](PropertiesBox/UIPropertiesBox.cpp.md) · [`UIFixedScrollBar.cpp`](ScrollBar/UIFixedScrollBar.cpp.md) · [`UIScrollBar.cpp`](ScrollBar/UIScrollBar.cpp.md) · [`UIScrollBox.cpp`](ScrollBar/UIScrollBox.cpp.md) · _and 8 more_
**Tier floor** — T3: an enumeration. Nothing constrains it but the requirement that the
values be stable, because scripts name them.

## Purpose

Widgets do not call each other. A widget tells its message target "this happened to me" with
one of these identifiers plus an optional payload, and the target decides what it means.
This file is the entire vocabulary, and it is flat: one namespace shared by the generic
window events, every built-in control, and a long tail of game-specific screens that do not
live in this module at all.

The flatness is the design's one real cost. A message identifier carries no information
about which widget could have sent it, so a container that receives `LIST_ITEM_CLICKED` must
check the sender's identity anyway. A rebuild is free to make these per-widget event types;
what it must preserve is the *numbering as exported to scripts*, because shipped Lua compares
against the exported names.

## State

`Stateless.`

## `UIMessage`

**Contract** — an identifier in one flat enumeration. Grouped by originator, in this order:

```text
ENUM UIMessage
  # Generic window — raised by the tree walk in UIWindow
  LBUTTON_DOWN, RBUTTON_DOWN, CBUTTON_DOWN        # left / right / middle press
  LBUTTON_UP,   RBUTTON_UP,   CBUTTON_UP
  MOUSE_MOVE
  MOUSE_WHEEL_UP, MOUSE_WHEEL_DOWN, MOUSE_WHEEL_LEFT, MOUSE_WHEEL_RIGHT
  LBUTTON_DB_CLICK                                 # synthesised, never delivered raw
  KEY_PRESSED, KEY_RELEASED, KEY_HOLD
  MOUSE_CAPTURE_LOST, KEYBOARD_CAPTURE_LOST        # sent to the window being displaced
  FOCUS_RECEIVED, FOCUS_LOST                       # cursor entered / left, not navigation

  # Controls owned by this module
  BUTTON_CLICKED, BUTTON_DOWN
  TAB_CHANGED
  CHECK_BUTTON_SET, CHECK_BUTTON_RESET
  RADIOBUTTON_SET
  DRAG_DROP_ITEM_{DRAG, DROP, DB_CLICK, LBUTTON_CLICK, RBUTTON_CLICK, SELECTED, FOCUSED_UPDATE}
  SCROLLBOX_MOVE                                   # the thumb was dragged
  SCROLLBAR_VSCROLL, SCROLLBAR_HSCROLL, SCROLLBAR_NEEDUPDATE
  CHILD_CHANGED_SIZE                               # a scroll view must re-lay-out
  LIST_ITEM_{CLICKED, SELECT, UNSELECT, FOCUS_RECEIVED}
  PROPERTY_CLICKED                                 # a context-menu entry
  MESSAGE_BOX_{OK, YES, QUIT_WIN, QUIT_GAME, NO, CANCEL, COPY}_CLICKED

  # Game screens, which live in another module but share this vocabulary
  TALK_DIALOG_*, PDA_TASK_*, INVENTORY_*, MAP_*, EDIT_TEXT_COMMIT, EDIT_TEXT_CANCEL
  MAIN_MENU_RELOADED
```

**Invariants**

- The mouse-button and wheel identifiers double as *input actions* passed into the mouse
  dispatch, not only as notifications. The same enumeration is used for both directions,
  which is why `MOUSE_WHEEL_UP` and `MOUSE_WHEEL_DOWN` are also passed as the *scroll
  direction argument* to the scroll handler. A rebuild that separates input actions from
  notifications must keep the wheel direction meaningful.
- `FOCUS_RECEIVED` / `FOCUS_LOST` mean the *pointer* entered or left, and predate the
  navigation focus system in [`ui_focus.cpp`](ui_focus.cpp.md). Shipped scripts know them by
  their older names (`STATIC_FOCUS_RECEIVED` / `STATIC_FOCUS_LOST`) and the script export
  keeps both spellings pointing at the same value.
- The game-screen entries at the tail are declared here even though nothing in this module
  raises them. That is the cost of the flat namespace; a rebuild that splits the vocabulary
  per module must still keep one global numbering for the script side.

**Notes** — several inventory entries are marked in the source as third-party mod additions
that were merged in. They occupy numbers in the middle of the enumeration, so removing them
would renumber everything after and break scripts that hold literal values. Treat the
numbering as append-only.
