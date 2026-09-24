# src/xrUICore/TabControl — mutual exclusion

> A set of latching buttons of which exactly one is pushed, identified by name rather than by
> index, and doubling as a settings control.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The control that owns "exactly one of these is active". Its members are **tab buttons** —
four-state buttons carrying a string identifier — and the control keeps the pushed one's
identifier and the previously pushed one's, announcing every change as a
`(new, previous)` pair.

It is also a settings control, whose value is the active tab's identifier, which is how the
options screens store a page choice or a preset name.

The radio button in [`Buttons/`](../Buttons/README.md) is a tab button with a fixed graphic, so
a group of radio buttons *is* one of these.

## The load-bearing ideas

**Identity is a name, not a position.** Tabs are addressed, stored and reported by string
identifier. Index-based accessors exist for convenience but the identifier is what a screen and
a saved setting hold, so reordering tabs in a layout does not change a stored value.

**Exclusion is enforced on notification, not on press.** A tab button reports its press; the
control unpushes the others and pushes this one. A rebuild that makes each button unpush its
siblings directly will emit two changes for one press.

**Two accelerator layers, separately switchable.** The control can own accelerators that cycle
between tabs, and the buttons can own their own per-tab accelerators. They are enabled
independently because a screen that already binds those keys must be able to turn one layer off
without losing the other. Only the buttons' layer is on by default.

**Stepping wraps or stops, by request.** Moving to the next tab takes a direction and a loop
flag, because a gamepad shoulder button wants wrapping and an arrow key usually does not.

**The active colours are a pair per role.** Text and button colour each have an inactive and an
active value, applied across the whole group when the selection changes — so the look of "this
tab is current" is authored once on the control rather than on each button.

## The twins

| Twin | Role |
|---|---|
| [`UITabControl.cpp`](UITabControl.cpp.md) | The exclusive set: name-addressed tabs, the (new, previous) change announcement, the two accelerator layers, stepping, group-wide colours, and the settings protocol |
| [`UITabControl.h`](UITabControl.h.md) | Its declaration and the script-facing accessors |
| [`UITabButton.cpp`](UITabButton.cpp.md) · [`UITabButton.h`](UITabButton.h.md) | One member: a four-state button carrying a string identifier, reporting its press for the control to resolve |
