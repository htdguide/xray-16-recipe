# src/xrGame/ui/UIMessageBoxEx.cpp

> Makes the toolkit's message box into a modal dialog: it takes on the box's geometry, closes
> itself on any outcome, and republishes the outcome both as a callback and as a notification.

**Needs** — [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [`UIDialogHolder.h`](../UIDialogHolder.h.md) · [`xrUICore/MessageBox/UIMessageBox.h`](../../xrUICore/MessageBox/UIMessageBox.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md)
**Tier floor** — T3.

## Purpose

The toolkit's message box is a *widget*: it knows how to lay out a caption, a body and one to
three buttons according to a style named in data. It does not know how to be modal, how to be
pushed onto the screen stack, or when to go away. This file supplies exactly those three
things, and nothing else — which is why it is eighty lines.

## State

```text
RECORD ModalMessageBox EXTENDS Dialog
  box        : MessageBox           # named "msg_box"; the only child
  func_on_ok : optional<handler>
  func_on_no : optional<handler>
```

**Invariants** — the wrapper's rectangle is the box's rectangle, and the box is then moved to
the wrapper's origin. So the two are the same box on screen and the wrapper adds no chrome.
The *box* owns its geometry because the style decides it; the wrapper is told.

## `InitMessageBox`

**Contract** — build the inner box from a named style, declining if the style is unknown.
Adopt its position and size, move it to the origin, and bind the affirmative outcome to the
caller's handler. Bind the negative outcome as well, but **only for the three styles that
actually have a second button** — yes/no, quit-to-desktop and quit-to-menu.

**Notes** — binding the negative handler conditionally, rather than letting an unused binding
sit idle, is not an optimisation: a style with one button emits the affirmative outcome for
that button, and binding the negative one anyway would be harmless but misleading. The
decision worth carrying is that *which outcomes exist is a property of the style*, and the
style is data.

## `SendMessage`

**Contract** — dispatch the notification to the bound handlers, then, for any of the six
outcome notifications, close the dialog; and in every case re-send the notification upward as
coming from the *wrapper*, so a screen can listen to one dialog rather than to its internals.

**Notes** — closing happens for every outcome including cancel, so a message box never needs
its caller to dismiss it. The re-send is what makes the two mechanisms — a direct handler and
a notification — both available; screens that own several boxes use the notification and
distinguish by sender, screens with one use the handler.

## `NeedCursor`

**Contract** — a pointer is needed whenever the player is on keyboard and mouse, and otherwise
only for the four styles that contain something to type into or copy from: the address prompt,
the password prompt, the remote-admin login and the copyable notice.

**Notes** — this is the file's only real decision. On a gamepad the buttons are reachable by
directional navigation, so a pointer would be noise; a text field is not reachable that way,
so the pointer comes back. The cursor is explicitly *not* re-centred when it appears, because
recentring would move it off whatever the player was already pointing at underneath.

## The passthroughs

**Contract** — the body text, the host and password fields, and the address field are read and
written straight through to the inner box.
