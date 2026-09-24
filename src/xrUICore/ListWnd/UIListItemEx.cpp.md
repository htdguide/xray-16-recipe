# src/xrUICore/ListWnd/UIListItemEx.cpp

> Toggles its own texture tint between fully transparent and a configured selection colour when the owning list selects or unselects it.

**Needs** — [`UIListItemEx.h`](UIListItemEx.h.md) · [`UIListItem.h`](UIListItem.h.md) · [`UIMessages.h`](../UIMessages.h.md)
**Used by** — [`UIListItemEx.h`](UIListItemEx.h.md)
**Tier floor** — T3.

## Purpose

A list row that shows its selected state by tinting its own background rather than by
drawing a separate highlight. Selection reaches it as a message from the owning list; the
row answers by swapping its tint between fully transparent and a colour configured from
data.

Keeping the highlight inside the row is what lets rows of different shapes sit in one list
without the list knowing how any of them draws.


## State

```text
RECORD ListItemEx EXTENDS ListItem
  selection_colour : colour    # default: alpha 200 over a warm brown
  # the row's own texture colour is transparent when unselected
```

## Construction

**Contract** — sets the row's texture colour to fully transparent, so an unselected row shows
nothing of its own texture. The tint is applied by colouring that same texture, not by drawing
an extra quad — which means the row *must* have a texture loaded for selection to be visible
at all.

## `send_message`

**Contract** — intercepts the list's select and unselect messages and sets the texture colour
to the selection colour or to transparent respectively. Every other message is dropped —
deliberately, because this override does not chain to the base.

**Notes** — not chaining means a row of this type does not forward messages to its own
children. No shipped row of this type has children, so the consequence is latent rather than
observed; a rebuild should chain.

## `set_selection_color`

**Contract** — replaces the tint. Takes effect on the next selection, not immediately.

## `draw`

**Contract** — plain delegation to the base row. Present as an override only because a text
length-limiting pass used to live here; nothing remains of it.
