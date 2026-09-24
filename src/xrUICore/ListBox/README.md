# src/xrUICore/ListBox — the list of selectable rows

> A scroll view that knows its children are rows: it can manufacture them from its own
> defaults, select one, find one, and reorder them.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The **list box** is a scroll view with a row model on top. It manufactures uniform rows from
the list's authored defaults — height, font, colour, text alignment — so a caller adds a string
or an icon rather than building a widget; it forwards each row's selection and click
notifications upward as the *list's* own, so an owner binds to the list and not to every row;
and it translates the mouse wheel into a scroll step.

A **row** draws a stretched-line highlight only while selected, selects itself on press or on
gaining pointer focus, and appends its fields at a running right edge so a row is a
left-to-right sequence of text and icon cells rather than a fixed layout.

A **pass-through row** is the same row that performs its selection and then declares the press
unhandled, so the event keeps travelling to widgets behind it.

## The load-bearing ideas

**The list renames its children's events.** A row's notification is re-sent as the list's, with
the list as sender. That indirection is what lets a screen bind one handler to a list instead
of tracking rows it never created.

**Selection follows the pointer, not only the click.** A row selects itself when it gains
pointer focus, so hovering moves the selection. The shipped screens depend on this for
keyboard-free browsing; a rebuild that selects only on click feels different immediately.

**Fields accumulate.** A row places each added field at its current right edge, so the row's
layout is the order of addition. There is no column model.

**The pass-through variant exists because consuming a press is the default.** A row that must
both select itself and let the press reach a background handler cannot do it by ordering — it
has to lie about consumption. That is a deliberate, narrow escape hatch.

**Text rows are localized.** Adding a row by text runs the string through the localization
table; adding by an already-resolved string does not.

## The twins

| Twin | Role |
|---|---|
| [`UIListBox.cpp`](UIListBox.cpp.md) | Manufacturing rows from the list's defaults, renaming their notifications, and the wheel-to-scroll step |
| [`UIListBox.h`](UIListBox.h.md) | Its declaration: find, select, reorder |
| [`UIListBoxItem.cpp`](UIListBoxItem.cpp.md) | The row: selected-only highlight, selection on press or pointer focus, fields appended at the running right edge |
| [`UIListBoxItem.h`](UIListBoxItem.h.md) | Its declaration |
| [`UIListBoxItemMsgChain.cpp`](UIListBoxItemMsgChain.cpp.md) · [`UIListBoxItemMsgChain.h`](UIListBoxItemMsgChain.h.md) | The row that selects itself and then reports the press unhandled |
