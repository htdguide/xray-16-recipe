# src/xrEngine/edit_actions.cpp

> The handler chain behind one key of the text editor: try each link's modifier requirement in turn, run the first that matches.

**Needs** — [`edit_actions.h`](edit_actions.h.md) · [`line_edit_control.h`](line_edit_control.h.md) · [`xr_input.h`](xr_input.h.md)
**Used by** — [`edit_actions.h`](edit_actions.h.md)
**Tier floor** — T3: a guarded linked list of callbacks.

## Purpose

The text editor binds several meanings to one key, separated by modifier — plain `Delete`
deletes forward, `Ctrl+Delete` deletes a word, `Shift+Delete` cuts. This file is the
mechanism: each key owns a chain of links, each link carries a required modifier set and
something to do, and a press walks the chain until one link's requirement is met.

Three kinds of link, and that is the whole file.

## `base` — the chain link

**Contract** — holds the rest of the chain and, when it has nothing of its own to do, passes
the press along. Owns what follows it, so releasing the head releases the chain.

```text
FUNCTION on_press(link, control)
  IF link.next EXISTS
    link.next.on_press(control)
```

**Invariants** — a chain is walked front to back, and the front is the most recently
registered link. Registration therefore reverses priority: the *last* handler registered for
a key is tried *first*. The editor's initialization relies on this, registering the
unmodified handler before the modified ones so that the specific chords are tested before
the general fallback.

## `callback_base` — a guarded action

**Contract** — runs its callback and stops the walk when the control reports the required
modifier set held; otherwise passes the press along. A requirement of *none* is satisfied
unconditionally, which is what makes an unmodified handler the fallback at the end of a
chain.

```text
FUNCTION on_press(link, control)
  IF control.modifiers_held(link.required)
    link.callback()
    RETURN                      # consumed; the rest of the chain is not tried
  base.on_press(link, control)
```

**Notes** — "stops the walk" is the decision. Without it, `Ctrl+C` would both copy *and* run
whatever plain `C` does, which is to type a letter.

The requirement is a *set*, and the test is "any of these bits", so a handler asking for
"either shift" matches left or right. Handlers that must distinguish ask for the specific
side.

## `key_state_base` — a modifier that records itself

**Contract** — sets one modifier flag on the control and then continues along the chain
unconditionally. It never consumes the press.

```text
FUNCTION on_press(link, control)
  control.set_modifier(link.flag, true)
  IF link.rest EXISTS
    link.rest.on_press(control)
```

**Notes** — this exists because a modifier's own press arrives as an ordinary key event, and
the editor's view of which modifiers are held must be correct *during* that event, not only
after it is re-read from the platform at the end. The six modifier keys each get one of
these at the head of their chain.

It deliberately does not consume, so a chord registered on a modifier key itself — the
keyboard-layout switch is bound to `Ctrl+LeftShift` — still reaches its handler.

A rebuild that reads the platform's modifier state at the start of each event instead of
tracking it does not need this kind of link at all.
