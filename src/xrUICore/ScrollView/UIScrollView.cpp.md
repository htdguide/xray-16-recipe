# src/xrUICore/ScrollView/UIScrollView.cpp

> A vertically scrolling container: it stacks its children on a movable pad, clips them to its own rectangle, draws only the visible run, and keeps a scroll bar in sync with the pad's offset.

**Needs** — [`UIScrollView.h`](UIScrollView.h.md) · [`ScrollBar/UIScrollBar.h`](../ScrollBar/UIScrollBar.h.md) · [`ScrollBar/UIFixedScrollBar.h`](../ScrollBar/UIFixedScrollBar.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Callbacks/UIWndCallback.h`](../Callbacks/UIWndCallback.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`ui_base.h`](../ui_base.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/xr_input.h`](../../xrEngine/xr_input.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIScrollView.h`](UIScrollView.h.md)
**Tier floor** — T3: layout arithmetic, a clip rectangle and a visible-range cache.

## Purpose

This is the workhorse container of the whole UI: every inventory list, every dialogue
transcript, every task description and every options page is a scroll view. It solves four
problems that do not belong in a plain window.

**Children are laid out, not placed.** A caller adds a child and never gives it a position;
the view stacks them top to bottom with a configurable gap and four independent indents.

**A moving viewport.** The children live on an intermediate *pad* window whose position is the
negated scroll offset. Nothing in the tree walk knows about scrolling — moving one window
moves all the content, and each child's absolute rectangle stays correct for hit testing.

**Clipping and culling.** The view pushes a scissor rectangle before drawing and draws only
the contiguous run of children whose rectangles intersect it. The run's start and end are
cached and invalidated on every layout or scroll, which is what keeps a thousand-line
transcript cheap.

**Focus follows scroll and scroll follows focus.** On a gamepad the focused widget must stay
on screen, so the view scrolls to it; and when scrolling moved, the cursor is warped onto the
newly visible widget so that hover and focus agree.

## State

```text
RECORD ScrollView EXTENDS Window
  pad            : Window       # holds every item; its negated y is the scroll offset
  scroll_bar     : ScrollBar    # either variant
  left/right/up/down_indent : real
  vert_interval  : real         # gap between consecutive items
  flags          : set of { vert_flip, need_recalc, fixed_scroll_bar,
                            items_selectable, inverse_dir }
  visible_range  : (int, int)   # first and last drawn child index; (-1,-1) means unknown
  sort_function  : optional<comparison over two windows>
  scrollbar_profile : text
```

**Invariants**

- `need_recalc` is the single deferral point: every mutation sets it and every observer that
  needs correct geometry clears it by running the layout first. A rebuild that lays out
  eagerly will re-lay out once per added item.
- `visible_range` of `(-1, -1)` means "unknown, rescan"; it is invalidated by any layout and
  by any change of scroll position.
- The pad's vertical position is always the negation of the bar's scroll position, in the
  toolkit's virtual screen units. The bar's range is *not* in those units — see
  `Scroll2ViewV`.
- `items_selectable` gates the whole selection surface: with it off, the selection accessors
  answer as if there were no selection at all rather than failing.

## `InitScrollView`

**Contract** — creates the pad if absent, then resolves which scroll bar this view actually
gets, then sizes and places the bar down the right edge and derives its step and page from the
view's height. Must be called after the view has its own size.

```text
FUNCTION init_scroll_view()
  IF no pad THEN create pad, own it, attach it
  pad.position <- (0, 0)

  IF the bar we were given is a fixed-geometry bar
    IF its profile's art is missing
      log and discard it            # fall back to the ordinary bar
  IF no bar
    create an ordinary bar, own it, attach it
    subscribe to its vertical-scroll message

  initialize the bar at the view's right edge, view height tall, vertical,
    with the configured profile when there is one
  shift the bar left by its own width so it sits inside the view
  step_size <- max(1, view height / 10)
  page_size <- view height
```

**Notes** — the bar's step is a tenth of the viewport and its page is the whole viewport, both
measured in the same units the range will be. That is what makes a wheel notch move a tenth of
a screen regardless of how tall the content is.

The fixed-bar fallback is the reason `InitScrollBar` returns a boolean at all: the older games
lack the art, and a view must still work.

## `RecalcSize`

**Contract** — the layout. Optionally sorts the items, then stacks them from the top indent
downward, accumulating the pad's height and taking the pad's width from the widest item.
Finally pushes the new extent into the scroll bar and marks the visible range unknown.

```text
FUNCTION recalc_size()
  IF sort_function EXISTS THEN sort pad's children by it

  cursor <- (right_indent, vert_interval + up_indent)
  height <- up_indent + down_indent
  width  <- 0

  FOR EACH item IN pad's children, reversed when vert_flip
    item.position <- cursor
    cursor.y <- cursor.y + item.height + vert_interval
    height   <- height   + item.height + vert_interval
    width    <- max(width, item.width)

  pad.size <- (width, height)
  IF inverse_dir THEN pad.y <- view height - pad height   # bottom-aligned content
  update_scroll()
  need_recalc <- false ; visible_range <- unknown
```

**Notes** — three details are load-bearing and look like mistakes.

The first item's horizontal position is the *right* indent, not the left. The left indent
never positions anything; it only shrinks the width offered to children. Copy it.

The vertical gap is added *before* the first item and *after* the last, so the content has one
interval of padding at each end over and above the indents.

`vert_flip` reverses the visual order without reordering the list, so an append becomes a
prepend — which is how the message log grows upward. `inverse_dir` instead keeps the order and
pins the stack to the bottom of the viewport. They are different ideas and both exist.

## `UpdateScroll` and `Scroll2ViewV`

**Contract** — pushes the pad's extent into the bar's range and the pad's current offset into
the bar's position.

```text
FUNCTION update_scroll()
  bar.height <- view height
  bar.range  <- 0 .. floor(pad height * scroll_to_view_ratio)
  bar.position <- floor(-pad.y)

FUNCTION scroll_to_view_ratio() -> real
  RETURN (view height + up_indent + down_indent) / view height
```

**Notes** — the ratio is the correction that makes a drag land where the eye expects. The pad
is clipped to the viewport *minus* the vertical indents, so the visible window is shorter than
the bar's travel; inflating the range by that same proportion makes the thumb reach the end of
its track exactly when the last item reaches the bottom of the clipped area. Without it the
last indent's worth of content is unreachable. It is one line and it is the least obvious line
in the file.

## `Draw`

**Contract** — lays out if needed, pushes a scissor rectangle equal to the view's rectangle
inset by the top and bottom indents, draws the visible run of children, pops the scissor, then
draws the scroll bar *outside* the clip so a bar wider than the view is not cut.

```text
FUNCTION draw()
  IF need_recalc THEN recalc_size()
  clip <- absolute rect, top += up_indent, bottom -= down_indent
  push_scissor(clip)

  IF visible_range IS known
    FOR EACH item IN pad's children FROM visible_range.first TO visible_range.last
      IF item.visible THEN item.draw()
  ELSE
    FOR EACH item, index IN pad's children
      IF item's rect intersects clip
        record index as first if none recorded yet ; record it as last
        IF item.visible THEN item.draw()
      ELSE IF a first was already recorded
        BREAK                     # the stack is ordered, so we are past the window

  pop_scissor()
  IF need_show_scroll_bar THEN scroll_bar.draw()
```

**Notes** — the early break is what makes the scan cheap, and it is only sound because items
are stacked in order with no overlap. A rebuild that allows free placement inside a scroll
view loses this and must scan everything.

Note the two different visibility notions: a child outside the clip is *culled* (not drawn,
still laid out, still hit-testable through the pad), while a child whose own visible flag is
false is skipped but still occupies its slot in the stack and still counts toward the visible
range. Both are needed — the first is performance, the second is content.

A scroll view is asserted never to contain another scroll view. Nested scissors and nested
capture are not supported.

## `OnMouseAction`

**Contract** — after the children have refused the event: a wheel notch pages the bar in the
matching direction; a mouse move with the left button physically held drags the pad directly,
clamped so the content cannot be pulled past either end, and then re-syncs the bar. Any change
of position invalidates the visible range.

**Notes** — the drag reads the *physical* button state rather than tracking a press, so a drag
begun outside the view continues inside it. The pad clamp here is the only place the content
extent is enforced directly; everywhere else the bar's clamp does it.

## `ScrollToWindow`

**Contract** — scrolls so that a given child sits at a chosen fraction of the viewport height,
defaulting to the middle. Refuses, without failing, a window that is not a direct child of the
pad.

```text
FUNCTION scroll_to_window(item, center_ratio = 0.5)
  REQUIRE item.parent IS pad
  IF need_recalc THEN recalc_size()
  set_scroll_position(floor(up_indent + item.y - view height * center_ratio))
```

## `SendMessage` and `Update` — the focus/scroll coupling

**Contract** — two paths keep the focused widget on screen.

On a focus-received message, if the newly focused widget is the globally focused one, find the
pad-level item that contains it, and if that item is not already the selected one, scroll to
it; if the scroll position actually changed, warp the cursor onto that item so hover follows.

Per frame, when the cursor is over the pad and items are *not* selectable, do the same for
whatever is currently focused — warping the cursor onto the focused widget rather than the
item.

A child reporting that it changed size marks the layout dirty.

**Notes** — this is the gamepad story. With no mouse, focus moves by direction keys and the
view must chase it; the cursor warp exists so that the mouse-driven parts of the toolkit
(hover, hint, press) agree with where focus went. The condition that scrolling must have
actually changed prevents the warp from fighting a real mouse.

The two paths use different "already there" tests and warp different windows, which reads as
accretion rather than design.

## `AddWindow` / `RemoveWindow` / `Clear` / `GetItem` / `GetSize` / `Items` / `Empty`

**Contract** — the content surface. Adding optionally transfers ownership of the child to the
tree and marks the layout dirty; removing detaches and marks dirty; clearing detaches
everything, marks dirty and scrolls to the top. Item access is by index into the pad's child
list, answering nothing for an out-of-range index.

## `ScrollToBegin` / `ScrollToEnd` / `SetScrollPos` / `GetCurrentScrollPos` / `GetMinScrollPos` / `GetMaxScrollPos`

**Contract** — the scroll surface, each laying out first if needed, clamping into the bar's
range, and then applying the offset to the pad through the same path the bar's own message
takes.

## `SetSelected` / `GetSelected` / `SelectFirst`

**Contract** — selection, only when the view was configured with selectable items. Setting
marks exactly one item selected and the rest not; reading returns the marked one; selecting
the first scrolls to the top, marks the first item, and hands it the focus if it is
registered for focus navigation. Every item is assumed to implement the selectable interface —
an unchecked assumption that fails loudly if a plain window is added to a selectable view.

## `GetDesiredChildWidth` / `UpdateChildrenLenght` / `GetHorizIndent` / `GetVertIndent` / `NeedShowScrollBar`

**Contract** — the width offered to a child is the view's width less both horizontal indents,
less the bar's width when the bar is shown. A fixed-geometry bar is *always* shown; an
ordinary bar only when the content is taller than the viewport. `UpdateChildrenLenght` pushes
that width onto every child, which is how a text item knows how wide to wrap before it can
measure its own height.

**Notes** — the ordering constraint this creates is the one newcomers trip on: a text child
must be given its width, then asked to fit its height, then added — and the helper the header
declares for exactly that sequence exists because getting it wrong yields items of height zero.

## `SetScrollBarProfile` / `SetFixedScrollBar` / `ForceUpdate` / `GetPadSize` / `ScrollBar`

**Contract** — configuration and accessors. The profile and the fixed-bar flag must be set
before initialization, since that is when the bar is built. `ForceUpdate` marks the layout
dirty from outside; `GetPadSize` lays out first and then answers the content extent.

## `FillDebugInfo`

**Contract** — exposes the indents, the interval, the five flags, the profile name and the
cached visible range to the debug overlay, with a button that forces a re-layout. Non-shipping
builds only.
