# src/xrGame/ui/UILogsWnd.cpp

> The PDA's log tab: one game day at a time, filtered by kind, rebuilt from the actor's news registry into recycled row widgets, and fed into the list in bounded batches so a busy day does not stall a frame.

**Needs** — [`UILogsWnd.h`](UILogsWnd.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UINewsItemWnd.h`](UINewsItemWnd.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/ScrollBar/UIFixedScrollBar.h`](../../xrUICore/ScrollBar/UIFixedScrollBar.h.md) · [`xrUICore/Buttons/UICheckButton.h`](../../xrUICore/Buttons/UICheckButton.h.md) · [`game_news.h`](../game_news.h.md) · [`alife_registry_wrappers.h`](../alife_registry_wrappers.h.md) · [`Actor.h`](../Actor.h.md) · [`date_time.h`](../date_time.h.md)
**Used by** — [`UILogsWnd.h`](UILogsWnd.h.md)
**Tier floor** — T3.

## Purpose

Everything the player has been told, browsable by day. Three decisions make it more than a
list: **the unit of browsing is one game day**, **rows are recycled rather than rebuilt**, and
**the refill is deferred and batched** so that a day with three hundred entries does not
produce a frame that takes a second.

## State

```text
RECORD LogTab EXTENDS Window
  selected_day    : timestamp       # always midnight; the day being shown
  first_day       : timestamp       # the day the game started; the lower bound
  filter_news     : bool            # check box
  filter_talk     : bool            # check box
  list            : ScrollView
  row_cache       : list<Widget>    # detached rows kept for reuse
  rows_pending    : list<Widget>    # filled rows not yet in the list
  queue           : list<int>       # indices into the news registry, still to build
  needs_reload    : bool
  ctrl_held       : bool
  layout          : Document        # kept, because rows are built from it lazily
```

**Invariants**

- `selected_day` is always **truncated to a day boundary**, and every navigation goes through
  the same truncate-and-shift helper, so the two ends of a period always agree.
- Navigation is clamped: never before the day the game started, never after the current day.
- The layout document is **kept for the tab's whole life**, because a row is constructed from
  it whenever the cache is empty.

## Truncating to a day

```text
FUNCTION day_boundary(t, shift_days) -> timestamp
  RETURN t - (t modulo one_day) + shift_days * one_day
```

**Notes** — one day in milliseconds is the unit of the whole tab: the period label, the two
navigation buttons, the filter range, and the bounds check. Game time and wall time share the
same representation, so this works unchanged at any time acceleration.

## The refill

**Contract** — a reload is *requested* by a flag and performed at the next update; it never
happens inline.

```text
FUNCTION reload()
  queue := empty
  IF there is no actor THEN clear the flag; RETURN       # opened before the level exists
  show the selected day as the period label; right-align its caption against it
  window := [selected_day, selected_day + one day]
  FOR EACH entry, by index, IN the actor's news registry
    IF the entry's kind passes its filter AND its time is inside the window
      THEN append its index to the queue
  clear the flag
  move every row currently in the list into the cache, detached and no longer self-deleting
  build the first batch

FUNCTION build_batch()
  take up to 30 entries off the BACK of the queue
  for each, take a row from the cache or build one, fill it, and append to rows_pending

FUNCTION update()
  IF a reload is pending THEN reload
  refresh the clock at most once a second, right-aligning its caption against it
  move every pending row into the list, in REVERSE order
```

**Notes** — four things here are decisions.

*The queue is consumed from the back and the pending rows are inserted in reverse*, so the
list comes out in **registry order** — oldest first. Two reversals that cancel; getting one of
them wrong silently reverses the log.

*Thirty rows per batch.* The number is arbitrary in magnitude and not in intent: it bounds the
work one frame does. As written, only the first batch is built during a reload and nothing
calls for the next, so a day with more than thirty entries shows only thirty — the batching
entry point is public and is meant to be pumped by the PDA's own work scheduler. Recorded as
an incomplete mechanism rather than a defect, because the entry point exists.

*Rows are recycled, not destroyed.* Moving them to the cache detaches them and clears their
self-delete flag; refilling takes them back. A row is a composite widget with its own text
layout, and rebuilding one per entry per day change was the cost this avoids.

*The two captions are right-aligned by measurement*, each frame for the clock and each reload
for the period: the caption is fitted to its text and then positioned so its right edge sits a
fixed gap left of the value. That is how a localized caption of any length stays attached to
its value.

## Filters and navigation

**Contract** — the two check boxes select which kinds of entry are shown, and both start on.
Toggling either, or moving a day, only sets the reload flag; the work happens on the next
update.

**Notes** — deferring through a flag means several changes in one frame — a filter toggle and
a day shift — cost one rebuild. It also means the handlers are trivial, which is why they can
be bound generically.

Navigation compares the day before and after clamping and **only requests a reload if it
actually changed**, so holding the button at either end of the range costs nothing.

## Keyboard scrolling

**Contract** — arrow keys scroll by **one unit**, page keys by the list's own page step, and
either page key with a control key held jumps to the start or the end. Arrow and page keys
also repeat while held.

**Notes** — the fine scroll is implemented by temporarily setting the scroll bar's step to one
and restoring it. That is a hack around the bar having a single step size, and it is worth
recording because the alternative — two step sizes on the bar — is the right fix and changes
chapter 15's scroll-bar contract.

The control-key state is tracked **by this window**, from key press and release events, rather
than read from the input layer. It is cleared by any key that is not one of the four scroll
keys or a control key, which means an unrelated keystroke between pressing control and
pressing page-down loses the modifier.

## Per-game variation

**Contract** — the tab is absent when its layout document is; the background is looked up as a
nine-slice frame and then as a frame line; the centre background as a frame and then as a
picture; the character portrait, the clock and its caption are each optional, with the clock
and its caption required *together*.

**Notes** — the same three-games idioms as the faction-war page, including the paired
requirement. The centre caption is composed by **appending a localized string to the text the
layout already set**, which is how a shipped document supplies a prefix or a symbol and the
engine supplies the translated noun.
