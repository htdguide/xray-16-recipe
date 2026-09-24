# src/xrUICore/ListWnd — the row-paged list

> A list of fixed-height rows that shows a whole number of them, scrolls by whole rows, and
> tracks the focused row separately from the selected one.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The other list in the chapter, and the differences from [`ListBox/`](../ListBox/README.md) are
all deliberate. Rows here are a **fixed height** known in advance, so the window shows an exact
whole number of them and scrolling is by row index rather than by pixel offset. Rows outside
that window are **hidden, not clipped**, so nothing partial is ever drawn and no scissor is
needed.

A **row** is a button carrying an index, a group identifier, an integer value and an opaque
payload — the group identifier is what lets several widgets act as one row. A second row
variant shows selection as a tinted background rather than as highlighted text.

## The load-bearing ideas

**Focused and selected are two different states.** The focused row is where keyboard and
gamepad navigation currently is; the selected row is the one the screen acts on. Conflating
them means arrow keys commit a choice, which is not how the shipped screens behave.

**A row is a group, not a widget.** Click and focus are propagated across every widget sharing
a row's group identifier, so a row made of a label, an icon and a value behaves as one target.

**Directional navigation scrolls rather than stopping.** Moving focus past the last visible row
scrolls the visible window by one row instead of refusing, which is what makes a long list
navigable without a pointer.

**Rows arrive two ways.** The list can manufacture a row from a description, or adopt one the
caller built. Both paths renumber every row's index and refresh the scroll bar afterwards,
which is the invariant that keeps index-addressed lookups correct after any insertion.

**Hidden is cheaper than clipped**, and is only available because the heights are uniform. The
row-index window is computed arithmetically; there is no intersection test.

## The twins

| Twin | Role |
|---|---|
| [`UIListWnd.cpp`](UIListWnd.cpp.md) | The whole-row window, hiding rather than clipping, group-wide click and focus propagation, and directional navigation that scrolls |
| [`UIListWnd.h`](UIListWnd.h.md) | Its declaration, with focused and selected tracked separately |
| [`UIListWnd_inline.h`](UIListWnd_inline.h.md) | The two insertion paths and the renumbering and scroll-bar refresh they share |
| [`UIListItem.cpp`](UIListItem.cpp.md) · [`UIListItem.h`](UIListItem.h.md) | A row: an ungrouped, auto-deleted button carrying index, group, value and payload, with its label indented past its icon |
| [`UIListItemEx.cpp`](UIListItemEx.cpp.md) · [`UIListItemEx.h`](UIListItemEx.h.md) | The row that shows selection as a tinted background rather than highlighted text |
