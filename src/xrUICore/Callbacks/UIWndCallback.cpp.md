# src/xrUICore/Callbacks/UIWndCallback.cpp

> A flat list of (widget or widget name, event, handler) bindings, matched first-wins on each incoming event.

**Needs** — [`UIWndCallback.h`](UIWndCallback.h.md) · [`callback_info.h`](callback_info.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIWndCallback.h`](UIWndCallback.h.md)
**Tier floor** — T3: a linear scan over a small list; the only constraint is that it must be able to hold a script closure alongside a native one.

## Purpose

Screens are built from XML and therefore do not hold pointers to most of their own widgets.
This file is how a screen says "when the widget called `btn_ok` reports a click, run this",
where the widget is identified by the name the data gave it.

## State

```text
RECORD Binding
  script_handler : optional<script callable>   # fires with no arguments
  native_handler : optional<function(Window, opaque)>
  widget         : optional<Window>   # bind by identity...
  widget_name    : text               # ...or by name, when widget is none
  event          : int
```

**Invariants**

- Exactly one of identity or name is meaningful per binding: a binding created by identity
  stores the literal name `"noname"` and is never matched by name; a binding created by name
  stores no pointer.
- The list is append-only for the life of the screen and is released with it.

## `on_event`

**Contract** — finds the *first* binding whose event matches and whose widget matches, then
runs its script handler (always, even when unset — an unset script callable is a no-op) and
its native handler (only when set). Does nothing at all when no binding matches, and never
walks past the first match.

```text
FUNCTION on_event(sender, event, data)
  IF sender IS none RETURN
  binding <- first b IN bindings WHERE
      b.event == event AND
      (IF b.widget EXISTS THEN b.widget == sender
                          ELSE b.widget_name == sender.name)
  IF binding IS none RETURN
  binding.script_handler()          # no arguments reach the script side
  IF binding.native_handler EXISTS
    binding.native_handler(sender, data)
```

**Notes** — first-wins means a screen cannot register two handlers for the same pair; the
second is dead. Matching by name means two widgets with the same name are indistinguishable,
which is a real hazard because the XML loader does not enforce unique names. Both are
observable behaviours a rebuild should keep, since the shipped screens were written against
them.

The script handler takes no arguments while the native one takes the sender and the event's
payload. That asymmetry is the binding seam's doing, not a decision of this file: script
callbacks in the game layer receive their context by capture instead.

## `register`

**Contract** — sets the given child's message target to this screen. A screen that inherits
this mixin must also be a window; the call resolves that at run time and would produce a null
target for a non-window user.

## `add_callback` / `add_callback_str`

**Contract** — append a binding, by widget identity or by widget name respectively. No
duplicate check, no removal.
