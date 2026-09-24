# src/xrUICore/Callbacks/callback_info.h

> The binding record and the predicate that matches an incoming event against it.

**Needs** — [`UIWndCallback.h`](UIWndCallback.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIScriptWnd.cpp`](../../xrGame/ui/UIScriptWnd.cpp.md) · [`UIWndCallback.cpp`](UIWndCallback.cpp.md) · [`UIWndCallback.h`](UIWndCallback.h.md)
**Tier floor** — T3.

## Purpose

Exists only so that the binding record can carry a script callable without the callback
mixin's own header pulling in the script engine. The split is arbitrary in a rebuild whose
script values are ordinary values — merge it into the mixin.

## State

```text
RECORD Binding
  script_handler : script callable of no arguments
  native_handler : function(Window, opaque)
  widget         : optional<Window>    # none when bound by name
  widget_name    : text
  event          : int                 # -1 when unset; a real event id otherwise
```

## `event_comparer`

**Contract** — the match predicate, carrying the sender and the event being dispatched.
A binding matches when its event equals the dispatched event *and* either its widget pointer
equals the sender, or — when it has no pointer — its stored name equals the sender's name.

```text
FUNCTION matches(binding) -> bool
  IF binding.event != event RETURN false
  IF binding.widget EXISTS RETURN binding.widget == sender
  RETURN binding.widget_name == sender.name
```
