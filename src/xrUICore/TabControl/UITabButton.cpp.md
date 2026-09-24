# src/xrUICore/TabControl/UITabButton.cpp

> A latching button that announces a tab change on press rather than on click, and that depresses or releases itself according to which tab the announcement names.

**Needs** — [`UITabButton.h`](UITabButton.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`Buttons/UIButton.h`](../Buttons/UIButton.h.md) · [`UIMessages.h`](../UIMessages.h.md)
**Used by** — [`UITabButton.h`](UITabButton.h.md)
**Tier floor** — T3: a two-line state rule over a broadcast message.

## Purpose

A tab is not a button that got latched; it is a button whose *press machine has been removed*.
The ordinary press/drift/release machine exists so that a click can be taken back by dragging
off the widget. A tab must not be takeable back — the page switches the moment the player
presses — and, more importantly, exactly one tab in a group is depressed at any time, which is
a fact about the group, not about any single tab.

So this file does two things: it bypasses the press machine, and it makes every tab listen to
the same tab-changed announcement and set its own state from it.

## State

```text
RECORD TabButton EXTENDS ThreeStateButton
  id                : text     # this tab's name, matched against the announcement
  id_was_defaulted  : bool     # the id was invented at load, not authored
```

**Invariants** — the id must be non-empty by the time the tab joins a control; the control
asserts it. A tab whose id was invented rather than authored is flagged so the XML reader can
tell the two cases apart.

## `OnMouseAction`

**Contract** — skips the button's press machine entirely and runs the plain window walk, so
children still get their chance and hover still updates, but no press state is ever entered
from the mouse path.

**Notes** — this is the whole reason the type exists. A rebuild that keeps one button type
with a "latching" flag must still suppress the drift-off-and-back behaviour, or a tab can be
pressed and cancelled.

## `OnMouseDown`

**Contract** — a left press raises the tab-changed message naming this tab and consumes the
press. Any other button is refused. Note that this happens on *press*, with no release
required and no notion of the pointer still being over the tab.

## `SendMessage`

**Contract** — every tab in a group receives the tab-changed announcement, including the one
that caused it, and each sets its own state from it: the named tab depresses and fires its
click, every other tab returns to normal. A disabled tab ignores the announcement entirely and
keeps whatever state it had.

```text
FUNCTION send_message(sender, message)
  IF NOT enabled RETURN
  IF message == TAB_CHANGED
    IF sender IS self
      press_state <- pushed
      on_click()          # the ordinary click notification, so callers see one event
    ELSE
      press_state <- normal
```

**Notes** — routing the group's exclusivity through a broadcast rather than through the
control's own bookkeeping is the design: no tab knows the group, and the control does not have
to know how a tab renders its state. The cost is that a *disabled* tab is skipped, so
disabling the active tab leaves it drawn as depressed while another tab is also depressed.
That is observable and is worth deciding about deliberately in a rebuild rather than
inheriting.

The click notification fires here, on the state change, rather than on the press — so a tab
activated programmatically notifies exactly as one activated by the player.
