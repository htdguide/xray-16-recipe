# src/xrGame/ui/UIMapWnd2.cpp

> The map screen's navigation cluster: nine buttons in a grid, built from whichever of two
> layout shapes the shipped document uses, with the four pan buttons driven by polling rather
> than by clicks.

**Needs** — [`UIMapWnd.h`](UIMapWnd.h.md) · [`UIMap.h`](UIMap.h.md) · [`UITaskWnd.h`](UITaskWnd.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md)
**Used by** — [`UIMapWnd.h`](UIMapWnd.h.md)
**Tier floor** — T3.

## Purpose

A continuation of [`UIMapWnd.cpp`](UIMapWnd.cpp.md) holding one cohesive piece: the nine-button
navigation pad. The split is a build-time convenience and carries no design meaning; a rebuild
merges it.

## `init_xml_nav`

**Contract** — build the navigation cluster in one of two shapes. When the document defines a
cluster parent element, create all nine buttons as its children, named by index. When it does
not — the first game's layout — create only four of them, individually named, as children of
the map header. Then bind the five buttons that act on press: legend, zoom in, centre on
actor, zoom out, zoom reset.

```text
FUNCTION build_nav_cluster(doc, root)
  nav_parent <- optional element "btn_nav_parent"
  IF nav_parent EXISTS THEN
    FOR i IN 0..8: nav_buttons[i] <- element "btn_nav_parent:btn_nav_<i>"
  ELSE
    bar <- root + ":main_wnd:map_header_frame_line:tool_bar"
    nav_buttons[zoom_reset] <- optional element bar + ":global_map_btn"
    nav_buttons[actor]      <- optional element bar + ":actor_btn"
    nav_buttons[zoom_more]  <- optional element bar + ":zoom_in_btn"
    nav_buttons[zoom_less]  <- optional element bar + ":zoom_out_btn"

  bind press of nav_buttons[legend]      -> toggle the legend on the owning task screen
  bind press of nav_buttons[zoom_more]   -> zoom in
  bind press of nav_buttons[actor]       -> centre on the actor
  bind press of nav_buttons[zoom_less]   -> zoom out
  bind press of nav_buttons[zoom_reset]  -> show the whole world map
```

**Notes**

- **The button index is its position in a three-by-three grid**: legend, up, zoom-in on the top
  row; left, actor, right in the middle; zoom-out, down, zoom-reset on the bottom. The middle
  column is the pan cross and the diagonals are the view commands. That is why the
  enumeration's order looks arbitrary in the header and is not — it is the layout.
- The four pan buttons are **not** bound to a press. They are polled instead; see below.
- The first game's shape has no pan buttons at all, so five of the nine are absent and every
  use is guarded.

## `UpdateNav` — the polled pan

**Contract** — at most once every ten milliseconds, if the cursor is over one of the four pan
buttons *and* that button is held down, pan the map one step in its direction. Only one button
acts per tick.

**Notes** — a press notification fires once, and a pan button must repeat while held; the
toolkit has no auto-repeat, so the screen polls the button's own state machine instead. The
ten-millisecond gate turns that poll into a repeat rate of roughly a hundred steps a second,
independent of frame rate, which is the only reason the pan speed is consistent. The cursor-over
test is what stops a button that was pressed and then dragged off from continuing to pan.

## The five view handlers

**Contract** — one-line delegations: zoom in, zoom out, centre on the actor, show the world
map, and — for the legend — reach *up* to the owning task screen and toggle its legend panel.

**Notes** — the legend handler is the one place the map screen knows what contains it. It
casts its parent to the task screen and does nothing when the parent is something else, which
is how the same map screen serves both the PDA's map page and the task page.
