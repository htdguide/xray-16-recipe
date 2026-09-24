# src/xrUICore/InteractiveBackground/UIInteractiveBackground.h

> A container of four alternative backgrounds — enabled, disabled, highlighted, touched — of which exactly one is drawn, over any widget type that can load a texture.

**Needs** — [`UI_IB_Static.h`](UI_IB_Static.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md)
**Used by** — [`UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`UIComboBox.h`](../ComboBox/UIComboBox.h.md) · [`UI_IB_Static.cpp`](UI_IB_Static.cpp.md) · [`UI_IB_Static.h`](UI_IB_Static.h.md)
**Tier floor** — T3: selection over a fixed-size slot array; the parametrization over the background widget type is incidental.

## Purpose

Has no implementation file: it is the substance. Four-state appearance is wanted by buttons,
by combo box frames, by track bars and by tab buttons, and those want different *kinds* of
background — a plain quad, a stretched three-segment line, a nine-slice frame. This file
factors the state machinery out of the widget kind.

The parametrization by widget kind is incidental C++. What survives is: the background is a
set of alternative child widgets, all the same size and position, of which one is current,
and the current one is the only one drawn.

## State

```text
RECORD InteractiveBackground
  states : map<State, optional<Widget>>   # slots: enabled, disabled, highlighted, touched
  current: optional<Widget>               # an alias of one of the slots, not a fifth widget

ENUM State { enabled, disabled, highlighted, touched, current }
```

**Invariants**

- `current` is always one of the four slots or the enabled slot; it is never independently
  owned. Selecting a state that was never loaded falls back to the enabled slot.
- Every loaded slot sits at the container's origin at the container's full size. The
  container is a window in the tree, so it inherits its parent's coordinate space; the slots
  therefore all coincide.
- The slots are ordinary children with auto-delete, so they are released with the container
  and they are visited by the tree walk — but `draw` is overridden so only the current one is
  actually drawn.

**Notes** — the "current" entry sharing the state enumeration with the four real states is
the reason the enum has five members and a total; it is a slot index, not a state a widget
can be in.

## `init_ib(pos, size)` / `init_ib(texture, pos, size)`

**Contract** — sets the container's rect; the second form additionally loads the enabled slot
from a texture. Neither touches the other three slots.

## `init_state(state, texture, fatal)`

**Contract** — creates the slot's widget if it does not exist, loads the named texture into
it, places it at the origin at the container's current size, and makes it current. Reports
whether the texture loaded. A missing texture is fatal or merely reported depending on the
flag, which is how a caller probes for one game's texture name and falls back to another's.

**Invariants** — the container's size must already be set when this runs, because the slot is
sized from it and is never resized afterwards except through the width/height overrides below.

**Notes** — making the just-loaded slot current is a side effect that the caller rarely wants;
the loading sequence ends with whichever state was loaded last showing. Every user immediately
re-selects on its first update, which hides it.

## `set_current_state(state)`

**Contract** — points `current` at the named slot, or at the enabled slot when that slot is
empty. Never fails, never draws.

## `draw`

**Contract** — draws only the current slot. Deliberately does *not* call the window base's
child walk, so the other three slots are never drawn even though they are children.

## `set_width` / `set_height`

**Contract** — forwards the new dimension to every loaded slot, including the ones not
current, so that a later state change does not reveal a stale size.
