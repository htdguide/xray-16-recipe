# src/xrUICore/ListWnd/UIListWnd_inline.h

> The two row-insertion paths of the paged list — one that manufactures a row from a description, one that adopts a row the caller built — and the index renumbering and scroll-bar refresh they share.

**Needs** — [`UIListWnd.h`](UIListWnd.h.md) · [`UIListItem.h`](UIListItem.h.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md)
**Used by** — [`UIListWnd.cpp`](UIListWnd.cpp.md) · [`UIListWnd.h`](UIListWnd.h.md)
**Tier floor** — T3.

## Purpose

A separate file only because the row type is a caller-supplied parameter in the original. The
decisions are substantive and belong to the list, not to the row type: where a new row lands,
what happens to the indices of the rows after it, and how the scroll bar is re-ranged.

## `add_item(description)`

**Contract** — manufactures a row of the caller's chosen type, initializes it from a text, a
horizontal shift, the slot the row count implies, and the list's row size; gives it the
caller's payload and value and the list's text colour; then inserts it.

```text
FUNCTION slot_y(row_number) -> real
  RETURN IF vert_flip THEN height - row_number * row_height - row_height
                      ELSE row_number * row_height
```

**Notes** — the slot is computed from the *current row count*, i.e. the row is positioned as if
appended, even when it is about to be inserted in the middle. The re-layout that follows the
insert corrects it. The `shift` argument is the row type's own affair — typically a left
indent for a nested entry.

## `add_item(row)`

**Contract** — attaches the row as a child, places it in the slot the row count implies at the
list's row size, inserts it into the row list at the requested position, renumbers, re-lays
out, and re-ranges the scroll bar.

```text
FUNCTION add_item(row, insert_before)
  attach_child(row)
  row.init_list_item(at slot_y(item_count), size = (item_width, item_height))

  IF insert_before == -1
    append row ; row.index <- count - 1
  ELSE
    REQUIRE insert_before <= count
    FOR EACH existing row FROM insert_before ONWARDS
      existing.index <- existing.index + 1     # make room
    insert row AT insert_before ; row.index <- insert_before

  relayout()                                   # re-place every visible row

  scroll.range      <- (0, count - 1)
  scroll.page_size  <- min(rows_that_fit, count)
  scroll.position   <- first_shown_index
  refresh_scroll_visibility()
```

**Invariants** — the row's index must equal its position in the list after every insertion and
removal; every lookup by index and every highlight comparison depends on it. Setting the index
also resets the row's group identifier to the same value, so an inserted row leaves its group
unless the caller re-groups it afterwards — a real consequence of the insert path that shipped
callers work around by grouping after inserting.

The scroll range is `count - 1`, not `count`: the scroll position is the index of the first
visible row and the page size is the number of visible rows, so the usable range is handled by
the scroll bar's own page-size arithmetic rather than by the range.
