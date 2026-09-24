# src/xrUICore/Hint/UIHint.cpp

> A tooltip box that fits itself to its text and is nudged to stay on screen near the cursor, plus the dwell rule that decides when a widget asks for it.

**Needs** — [`UIHint.h`](UIHint.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`xrEngine/StringTable/StringTable.h`](../../xrEngine/StringTable/StringTable.h.md)
**Used by** — [`UIHint.h`](UIHint.h.md)
**Tier floor** — T3.

## Purpose

Two types with one job between them: the box knows how to size and place itself, and the
mixin knows when a widget wants one.

## State

```text
RECORD Hint EXTENDS Window
  background : FrameWindow      # nine-slice
  text       : Static
  visible    : bool
  border     : real             # inset used when fitting near the cursor
  clip_rect  : rect             # defaults to the whole virtual screen

RECORD HintOwner EXTENDS Window          # the mixin
  hint       : optional<Hint>            # shared, not owned
  delay      : int (milliseconds)        # default 1000
  text       : text
  armed      : bool
```

**Invariants**

- The hint is *not* owned by the mixin. One box is shared by every hint-owning widget on a
  screen, and the last one to set the text wins. Nothing arbitrates, so two widgets whose
  rectangles overlap can fight for it frame by frame.
- `armed` is set when the pointer enters the widget and cleared whenever the hint is
  disabled, which happens on entering, on leaving and on any visibility change. Entering
  therefore *clears and re-arms*, which is what makes the dwell timer restart on each hover.

## `UIHint::init_from_xml`

**Contract** — initializes the box's own rect from the named element, then re-roots the
document at that element and builds a background child from a `background` sub-element and a
text child from a `text` sub-element, reads the border inset from the background element's
attribute, restores the document root, and starts hidden.

**Notes** — the temporary re-rooting is how the toolkit expresses relative paths in a flat
path-string API. A rebuild with a real node handle passes the node and deletes the save and
restore.

## `UIHint::set_text`

**Contract** — empty or absent text hides the box and returns. Otherwise: show it, set the
text, grow the text child's height to fit its wrapped content, and set both the background's
and the box's height to that plus a 20-unit margin. Width is never changed — the box's width
comes from the layout and the text wraps inside it.

## `UIHint::draw`

**Contract** — when visible, first reposition the box so it lies near the cursor and entirely
inside the clipping rectangle, then draw normally. The fit routine is shared with the rest of
the toolkit and lives in [`UIWindow.cpp`](../Windows/UIWindow.cpp.md): it tries the box above
the cursor, then shifted left, then below, then below-and-right clear of the cursor glyph,
then pushed up from the bottom edge. It refuses — and the box is drawn where it was — if the
cursor is not inside the clipping rectangle at all.

## `UIHintWindow::update_hint_text`

**Contract** — the dwell rule. Shows nothing unless the pointer is over the widget, the widget
has text, and the widget is armed; and nothing until the pointer has been over the widget for
longer than the configured delay, scaled by the simulation time factor. Complains once and
gives up if the widget was never given a hint box.

```text
FUNCTION update_hint_text()
  IF NOT cursor_over_window OR text IS empty OR NOT armed RETURN
  IF now < focus_receive_time + delay * time_scale RETURN
  IF hint IS none THEN log("owner has no hint window"); RETURN
  hint.set_text(text)
```

**Notes** — `focus_receive_time` here means "the moment the pointer entered", not keyboard
focus. The delay defaults to one second, against the 700 ms used by the button hint; the two
mechanisms were written years apart and neither constant is derived from anything.

## `UIHintWindow::on_focus_receive` / `on_focus_lost` / `show`

**Contract** — entering the widget clears the hint and arms the timer; leaving clears it and
disarms; a visibility change clears it. "Clear" means setting the shared box's text to
nothing, which hides it — so one widget leaving hides a hint another widget may have just
set. That is the shared-box hazard, and the reason a screen should not put two hint owners
under one pointer position.
