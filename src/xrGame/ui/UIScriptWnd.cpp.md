# src/xrGame/ui/UIScriptWnd.cpp

> The screen base class scripts subclass: a modal dialog that routes each child notification to a
> script function chosen by (sending widget, message id), falling back to normal propagation.

**Needs** — [`UIScriptWnd.h`](UIScriptWnd.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Callbacks/callback_info.h`](../../xrUICore/Callbacks/callback_info.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIScriptWnd.h`](UIScriptWnd.h.md)
**Tier floor** — T2: holds script function references whose lifetime must end with the screen

## Purpose

Chapter 15's toolkit delivers a widget's events to its *message target* as a
`(sender, message id, payload)` triple. This file is the adapter that turns that triple into a call
into a script function. A script screen declares its bindings once at construction and then never
writes an event dispatch again; everything the player does arrives as one of its registered
functions being called.

The file is deliberately thin: the interesting half — the class export, the script-overridable
methods, and the typed accessors — lives in
[`UIScriptWnd_script.cpp`](UIScriptWnd_script.cpp.md), separated because it must be compiled against
the script engine.

## State

```text
RECORD Binding
  control_name : text        # the widget's name, as set by Register
  event        : int (16-bit)  # a message id from the toolkit's flat vocabulary
  callback     : script function reference, optionally with a bound receiver

RECORD ScriptScreen extends ModalDialog
  bindings : list<Binding>   # owned; released with the screen
```

Invariants:

- Every binding holds a live reference into the script virtual machine for as long as the screen
  exists, so the whole list must be released when the screen is — dropping a screen without
  releasing its bindings leaks into the script heap.
- Matching is by `(sending widget, message id)`, and the **first** matching binding wins; later
  duplicates for the same pair are unreachable.

## `SendMessage`

**Contract** — The single dispatch point. Finds the first binding matching the sending widget and
the message id; if there is one, calls its script function and stops. If there is none, defers to
the base dialog's handling, which is what keeps unhandled messages propagating normally.

```text
FUNCTION SendMessage(sender, message_id, payload)
  match <- first binding in bindings matching (sender, message_id)
  IF match IS none THEN
    RETURN base.SendMessage(sender, message_id, payload)
  match.callback()          # the payload is not passed on
```

**Notes** — The payload is deliberately dropped: script handlers are called with no arguments and
recover context from the screen object they are methods of. A rebuild that passes the payload
through would still satisfy every shipped script, but would not be required by any.

Matching compares the *sending widget*, not the control name stored in the binding. The name in a
binding is resolved to a widget elsewhere in the callback machinery (chapter 15's binding table);
this file's job is only the lookup and the call.

## `Register`

**Contract** — Two forms. Both redirect the child's message target to this screen, so its
notifications arrive here instead of at its parent. The two-argument form also sets the child's
name, which is what a name-keyed binding will later match against. Registering is the act that opts
a widget into script handling; an unregistered child's events go to its parent as usual.

## `AddCallback`

**Contract** — Appends a binding of `(control name, message id)` to a script function. Two forms:
a plain function, and a function plus the object it should be called on, which is how a script binds
one of its own methods. Neither form checks for a duplicate pair, and neither validates that the
named control exists — a binding for a name no control carries is simply never matched.

```text
FUNCTION AddCallback(control_name, event, function [, receiver])
  b <- new Binding
  b.callback.set(function [, receiver])
  b.control_name <- control_name
  b.event        <- event
  bindings.append(b)
```

## `Load`

**Contract** — Takes a layout document name and returns success unconditionally, doing nothing. It
exists because it is part of the frozen script surface: shipped scripts call it, and a rebuild must
keep accepting the call. Screens load their own layout in their constructor instead.

**Notes** — A rebuild may implement this as a real load, but must not make it fail for a name that
the current implementation silently accepts.
