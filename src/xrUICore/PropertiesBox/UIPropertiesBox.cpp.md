# src/xrUICore/PropertiesBox/UIPropertiesBox.cpp

> The pop-up context menu — a framed list that places itself inside a bounding rectangle, takes mouse and keyboard exclusively while open, and closes on a click outside itself.

**Needs** — [`UIPropertiesBox.h`](UIPropertiesBox.h.md) · [`ListBox/UIListBox.h`](../ListBox/UIListBox.h.md) · [`ListBox/UIListBoxItem.h`](../ListBox/UIListBoxItem.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIPropertiesBox.h`](UIPropertiesBox.h.md)
**Tier floor** — T3: placement arithmetic and a capture protocol; no device or format contact.

## Purpose

The right-click menu that appears over an inventory slot or a map marker. Three problems make
it more than a list in a box: it must appear fully inside a caller-supplied rectangle from an
arbitrary anchor point, it must swallow every input while open so the screen underneath does
not also react, and it must support exactly one level of submenu whose lifetime is tangled
with its parent's.

## State

```text
RECORD PropertiesBox EXTENDS FrameWindow
  list            : ListBox        # owns the entries; immediate-selection mode
  sub_menu        : optional<PropertiesBox>   # the one box that may open beside this one
  parent_menu     : optional<PropertiesBox>   # who opened us; the reverse of the above
  last_show_rect  : Rect           # the bounding rect of the last open, reused by the submenu
  sub_initiator   : optional<Window>  # the entry the open submenu belongs to
```

**Invariants**

- `sub_menu.parent_menu` is this box whenever `sub_menu` exists. The pair is written at
  construction and never revised; the source marks the duplication as a hazard and a rebuild
  should store one edge and derive the other.
- A box must not be destroyed while its submenu is shown — the child hides the parent on the
  way out, so the parent must outlive it.
- While shown, the box holds the parent's mouse capture, the parent's keyboard capture, and
  the focus system's lock. `Hide` releases all three or the screen underneath stays deaf.
- The list is in *immediate selection* mode: hovering an entry selects it. There is no
  separate "click to select, click again to act" step, because a context menu acts on
  release.

## `InitPropertiesBox`

**Contract** — sizes and positions the box, attaches the list, and configures both from a
shipped layout element. Fatal if no layout is found.

```text
FUNCTION init_properties_box(position, size)
  place self at position with size
  attach list

  doc <- load ui layout "actor_menu"
  IF doc missing OR it has no "properties_box" element
    doc <- load ui layout "inventory_new"        # the older games put it here
    FAIL WITH "no properties_box element" IF still absent

  frame_texture <- doc["properties_box:texture"]   # fatal when absent
  init_frame_texture(frame_texture)
  configure list from doc["properties_box:list"]

  list.place at (5, 5) with size - (10, 10)        # a uniform inset for the frame art
```

**Notes** — the two-document fallback is the whole compatibility story for this widget: the
third game moved the element into the actor menu, so a rebuild must try the newer name first
and fall back silently, failing only when neither has it. The five-unit inset is the frame
art's border thickness in the toolkit's virtual screen space; it is a constant only because
every shipped frame texture uses the same border.

## `Show`

**Contract** — opens the box at an anchor point, constrained to a bounding rectangle, and
seizes input. The bounding rectangle is remembered for a later submenu. Takes mouse capture,
keyboard capture and the focus lock; resets the list's selection; selects the first entry when
the player is on a gamepad, because a gamepad has no resting cursor to hover with.

```text
FUNCTION show(bounds, anchor)
  last_show_rect <- bounds
  # prefer up-left of the anchor, then the other three quadrants,
  # and fall back to down-right even if it overflows
  IF fits left and down   THEN at (anchor.x - width, anchor.y)
  ELSE IF fits left and up   THEN at (anchor.x - width, anchor.y - height)
  ELSE IF fits right and up  THEN at (anchor.x,         anchor.y - height)
  ELSE                          at (anchor.x,         anchor.y)

  show and enable
  reset every child
  parent.capture_mouse(self)
  parent.capture_keyboard(self)
  focus.lock_to(self)
  list.reset()
  IF input device is a controller THEN list.select_first()
```

**Notes** — the quadrant order is "left and down" first, which puts the menu up and to the
left of the cursor. That is the opposite of the desktop convention and is what the shipped
inventory expects; a rebuild that flips it will place menus over the slot the player just
clicked.

## `Hide`

**Contract** — closes the box, drops its own mouse capturer, releases the parent's mouse and
keyboard captures if it still holds them, unlocks the focus system, and closes its submenu.
Idempotent enough to be called from a closing child.

**Notes** — the guarded releases matter: a box may be hidden while some other widget has since
taken capture, and clearing a capture it does not own would deafen that widget. The focus
unlock is *not* guarded by ownership — it unlocks whatever lock exists — which is a latent
bug a rebuild should fix by checking the locker is this box.

## `ShowSubMenu`

**Contract** — opens the child box beside the currently selected entry, choosing the side that
fits. Requires a child to exist and to be closed.

```text
FUNCTION show_sub_menu()
  sub_initiator <- list.selected_item
  bounds <- last_show_rect
  pos    <- self.position
  pos.y  <- pos.y + sub_initiator.position.y + sub_initiator.height / 2

  IF pos.x + self.width + sub_menu.width < bounds.right
    bounds.left <- pos.x                 # child opens to the right: forbid it drifting left
    pos.x       <- pos.x + self.width
  ELSE
    bounds.right <- pos.x                # child opens to the left: forbid it drifting right

  sub_menu.show(bounds, pos)
```

**Notes** — the trick is that the side is chosen here by *narrowing the bounding rectangle*
and then letting the child's own quadrant rule do the placement. The child is never told
which side it is on.

## `OnItemReceivedFocus`

**Contract** — a callback registered on every entry when this box has a submenu. When focus
lands on an entry other than the one the open submenu belongs to, the submenu closes. This is
what makes a submenu behave like a submenu: moving the pointer to a different row dismisses it.

## `AddItem`

**Contract** — appends a text entry, stamping it with the caller's tag and payload pointer;
returns success unconditionally. When this box has a submenu, the new entry is also
registered as a callback source for focus arrival.

**Notes** — the tag is the removal key and the payload is the caller's own identity for the
entry; the widget never interprets either. A rebuild that gives entries a typed value can
collapse both into one field.

## `OnMouseAction`

**Contract** — a left press outside the box closes it and consumes the event; a right press
outside closes it but lets the event continue; wheel events are swallowed so the screen behind
does not scroll. Everything else goes to the children.

**Notes** — the asymmetry between the two buttons is deliberate: a right click outside is
usually the player opening a context menu somewhere else, and that second open must still
happen.

## `OnKeyboardAction`

**Contract** — after the children have had the key, translate it through the UI binding table
(falling back to the gameplay table) and decide whether the key escapes the menu. Movement
keys and the four focus-navigation keys are let through; back and quit close the menu;
*everything else is consumed*.

**Notes** — the default-consume is the point. A context menu is modal for the keyboard, and
the two exceptions exist so the player can still walk while a menu is up and so the focus
system can still move the highlight. A rebuild that defaults to passing keys through will
find hotkeys firing behind an open menu.

## `AutoUpdateSize`

**Contract** — shrink-wraps the box around its entries: height from the entry height times the
count plus the list's vertical indent, width from the longest rendered entry plus the
horizontal indent plus two units. Then re-lays the children.

**Notes** — the two extra units are a rounding cushion for the measured text width; without
them the longest entry can clip by a pixel. The list is sized to the *whole* box here, not
inset like at construction, which means an auto-sized box loses its frame inset. Recorded as
observable behaviour rather than a defensible decision.

## `SendMessage`

**Contract** — when the list reports a click, forward it to the message target as a property
click, then close — but only if this box is the *last* box in the chain, and closing it also
closes its parent. A box that has a submenu stays open, because its entries open submenus
rather than choosing anything.

## `GetClickedItem` / `RemoveItemByTAG` / `RemoveAll` / `GetItemsCount` / `Update` / `Draw`

**Contract** — pure delegation to the list and to the framed window underneath.
