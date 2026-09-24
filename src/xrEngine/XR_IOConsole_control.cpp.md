# src/xrEngine/XR_IOConsole_control.cpp

> The two cursors — command history and suggestion selection — and the clamping that keeps them inside their lists.

**Needs** — [`XR_IOConsole.h`](XR_IOConsole.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: index arithmetic over two lists

## Purpose

The console has two things a person moves through with arrow keys: previously executed
commands, and completion suggestions. Both are an index into a list that may shrink under
them, so both need the same clamping treatment, and the suggestion cursor additionally
drags a *window* of visible rows behind it. This file is those two cursors and nothing else;
it exists as a separate file for readability rather than for any structural reason.

## State

`Stateless.` It moves the indices declared in [`XR_IOConsole.h`](XR_IOConsole.h.md).

## The history cursor

**Contract** — the history is stored oldest-first and browsed newest-first, so the cursor is
an offset *back* from the end. Index −1 means "not browsing"; zero is the most recent entry.
Appending trims the oldest entries beyond the maximum.

```text
FUNCTION add_to_history(console, line)
  IF line is empty
    RETURN
  history.append(line)
  WHILE history.size > 64
    history.remove_first()

FUNCTION older(console)              # "previous command", the up arrow
  history_index = history_index + 1
  IF history_index >= history.size
    history_index = history.size - 1

FUNCTION newer(console)              # "next command", the down arrow
  history_index = history_index - 1
  IF history_index < 0
    history_index = 0

FUNCTION reset(console)
  history_index = -1
```

**Notes** — the naming in the source is inverted with respect to what it does: the function
called *next* decrements and the one called *previous* increments. That is consistent once
you know the index counts backwards from the newest entry, and it is worth saying plainly
because it is the kind of thing a rebuild silently gets wrong.

Both clamp at the ends rather than wrapping: holding the up arrow stops at the oldest
command instead of cycling back to the newest, which is what a person expects from a
history.

After a reset the index is −1, which is *not* a valid list position. Anything that reads the
history must move the cursor first; `SelectCommand` (in
[`XR_IOConsole.cpp`](XR_IOConsole.cpp.md)) asserts the index is valid before using it,
which is why every caller pairs a move with a select.

## The suggestion cursor

**Contract** — a selected index plus the index of the first visible row. Moving the selection
drags the window only as far as needed to keep the selection inside it, so the list does not
scroll while the selection moves within the visible rows.

```text
FUNCTION select_next(console)
  selected = selected + 1
  clamp_forward(console)

FUNCTION clamp_forward(console)
  IF selected >= tips.size
    selected = tips.size - 1
  lowest_first = max(selected - VISIBLE_ROWS + 1, 0)
  IF lowest_first > first_visible
    first_visible = lowest_first        # scroll down only as far as required

FUNCTION select_previous(console)
  selected = selected - 1
  clamp_backward(console)

FUNCTION clamp_backward(console)
  IF selected < 0
    selected = 0
  IF first_visible > selected
    first_visible = selected            # scroll up only as far as required

FUNCTION reset_selection(console)
  selected = -1
  first_visible = 0
  tips_suppressed = false
```

**Notes** — `VISIBLE_ROWS` is 14, and it is the same number the page-up and page-down
actions move by (see [`XR_IOConsole_callback.cpp`](XR_IOConsole_callback.cpp.md)), so a page
move lands the previously-off-screen row exactly at the edge. Tying the page size to the
window size is the decision; the particular 14 is a display choice.

Resetting the selection also clears the suppression flag, which is what makes suggestions
reappear as soon as anything changes after a person dismissed them.

An empty suggestion list leaves the selection at −1 after a forward clamp, since the size
minus one is −1. That is correct and is relied on: "no suggestion selected" and "the list is
empty" are the same state as far as the Enter key is concerned.
