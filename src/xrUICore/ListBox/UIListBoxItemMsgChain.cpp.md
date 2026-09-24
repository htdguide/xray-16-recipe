# src/xrUICore/ListBox/UIListBoxItemMsgChain.cpp

> Runs the ordinary row's press behaviour and then reports the press as unhandled.

**Needs** — [`UIListBoxItemMsgChain.h`](UIListBoxItemMsgChain.h.md) · [`UIListBoxItem.h`](UIListBoxItem.h.md)
**Used by** — [`UIListBoxItemMsgChain.h`](UIListBoxItemMsgChain.h.md)
**Tier floor** — T3.

## Purpose

A list row that does its ordinary work on a press and then *declines* to consume the
event, so the press continues to the containing list. This exists because the toolkit's
event model stops at the first handler that claims an event: a row that both reacts and
lets its owner react cannot be expressed by handling or not handling, only by handling and
then reporting otherwise.

A rebuild whose event model distinguishes "observed" from "consumed" deletes this file and
sets a flag instead.


## `on_mouse_down`

**Contract** — performs the base row's press handling — selection, the selection and click
notifications — and then returns "not handled" regardless of what the base returned.

**Notes** — the window dispatch stops at the first child that reports handling a press, so
returning false is the entire mechanism: the press is offered to the next candidate as if this
row had ignored it. The consequence a rebuild must keep is that the row's *effects* still
happen; only the consumption is suppressed.
