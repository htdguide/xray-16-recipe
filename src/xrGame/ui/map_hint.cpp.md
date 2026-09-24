# src/xrGame/ui/map_hint.cpp

> The map's tooltip: one panel in two modes — a single wrapped line for an ordinary map spot, or a
> five-part summary for a task — each sizing the panel to its content.

**Needs** — [`map_hint.h`](map_hint.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`../map_location.h`](../map_location.h.md) · [`../map_spot.h`](../map_spot.h.md) · [`../GameTask.h`](../GameTask.h.md) · [`../GametaskManager.h`](../GametaskManager.h.md) · [`../Actor.h`](../Actor.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`map_hint.h`](map_hint.h.md)
**Tier floor** — T3: text measurement and vertical stacking

## Purpose

Hovering anything on the map produces a tooltip, and what it says depends on what was hovered: a
plain map spot gets its own one-line description, while a spot that belongs to a task gets the
task's icon, title, receipt time, remaining time and description. Rather than two panels, there is
one panel holding both sets of labels, with a mode switch showing one set and hiding the other.

The task mode is the file's substance, and it is almost entirely **vertical stacking**: each part is
placed below the one before it, measured after its text is set, and the panel is finally sized to
the last part's extent. Nothing is at an authored position except the first.

## State

```text
RECORD MapTooltip extends FramedWindow
  owner    : optional<Widget>            # whose tooltip this currently is
  border   : optional<FramedWindow>      # present only in the older layout
  labels   : map<name, Widget>           # "simple_text", "t_icon", "t_caption",
                                         # "t_time", "t_time_rem", "t_hint_text"
  icon_x, caption_x : real               # the two authored left margins, captured at build
```

Invariants:

- Exactly one mode's labels are shown at a time: `simple_text` alone, or the other five together.
- `icon_x` and `caption_x` are the panel's two left margins, captured once. Every horizontal
  placement in task mode is one or the other; there are no other x positions.
- The panel grows to its content on every fill and never shrinks below the authored minimum height.

## `Init`

**Contract** — Builds from either of two layout vocabularies. The newer games declare the whole
panel, with all six labels, under the caller's path. The oldest declares only a plain one-line
tooltip in a separate document, with its own frame and description.

```text
FUNCTION Init(document, path)
  item_document <- load "hint_item.xml" (optional)
  IF document[path] configures as a framed window THEN     # newer layout
    FOR EACH name IN (simple_text, t_icon, t_caption, t_time, t_time_rem, t_hint_text)
      labels[name] <- required static document[path + ":" + name]
    icon_x    <- labels["t_icon"].x
    caption_x <- labels["t_caption"].x
  ELSE                                                      # oldest layout
    REQUIRE item_document has "hint_item"; configure from it
    border                 <- framed window "hint_item:frame"
    labels["simple_text"]  <- static "hint_item:description"
```

**Notes** — Whether the newer path's configuration is allowed to fail depends on whether the older
document was found: if it was not, the newer path is required and a failure is fatal. That is what
makes the fallback safe — the panel refuses to silently produce nothing. Both documents missing is a
hard failure naming both paths.

## `SetInfoStr`

**Contract** — Plain mode. Shows only the one-line label, sets its text from a localization
identifier, wraps it, and sizes the panel around it.

```text
FUNCTION SetInfoStr(identifier)
  mode <- plain
  text <- labels["simple_text"]
  text.text <- localized(identifier)
  text.fit_height_to_text()
  new_width  <- text.x + text.width + 20
  new_height <- max(64, text.y + text.height + 20)
  IF border IS none THEN
    self.size <- (new_width, new_height)          # the panel width follows the text
  ELSE
    self.height <- new_height                     # the older layout keeps its authored width
    border.size <- self.size                      # and the frame follows the panel
```

**Notes** — The 20-unit padding and the 64-unit minimum height are authored. The two layouts differ
in whether the panel's width follows the text: the newer one shrinks to fit, the older one keeps its
authored width and only grows vertically, because its frame artwork is not horizontally stretchable
in the same way.

## `SetInfoMSpot`

**Contract** — Given a map spot, asks the task manager whether any task owns its location. If one
does, renders the task; otherwise renders the spot's own description line. This is the whole
dispatch between the two modes.

## `SetInfoTask`

**Contract** — Task mode. Fills the five labels, stacks them, then sizes the panel. Two layouts
again, chosen by whether the task's icon actually loaded.

```text
FUNCTION SetInfoTask(task)
  mode <- task
  icon_loaded <- labels["t_icon"].bind_texture(task.icon)
  labels["t_icon"].stretch <- true

  labels["t_caption"].text <- localized(task.title); fit height
  labels["t_time"].text    <- the task's receipt time, formatted as date and time
  labels["t_time"].y       <- caption.bottom + 7

  show_remaining <- task.receipt_time != task.deadline
  labels["t_time_rem"].shown <- show_remaining
  IF show_remaining THEN
    labels["t_time_rem"].text <- localized("ui_st_time_remains") + " "
                                 + the period from now to the deadline
  labels["t_time_rem"].y <- time.bottom + 7

  labels["t_hint_text"].text <- localized(task.description); fit height
  last <- t_time_rem IF show_remaining ELSE t_time
  labels["t_hint_text"].position <- (icon_x, last.bottom + 10)

  IF icon_loaded THEN
    show the icon at icon_x
    put the caption, the time and the remaining time at caption_x   # indented past the icon
    give the caption the time label's width
    push the description below the icon if the icon is taller
  ELSE
    hide the icon
    put the caption, the time and the remaining time at icon_x      # no indent
    give the caption the description's width

  self.size <- (description.right + 20, description.bottom + 20)
```

**Invariants** — A task with no deadline is recognised by its deadline equalling its receipt time,
and then the remaining-time line is hidden and the description moves up to take its place. The stack
is therefore four or five parts deep and the panel's height differs accordingly.

**Notes** — The two left margins are the whole difference between the icon and no-icon layouts: with
an icon the text is indented past it, without one it starts at the icon's margin. The caption's
width is borrowed from a *different* label in each case — the time label when indented, the
description when not — because those are the two labels whose authored widths match the available
space in each arrangement. It is an odd way to express "the remaining width", and copying it is what
reproduces the original's wrapping.

The 7-unit gaps between stacked lines and the 10-unit gap before the description are authored
spacing; the 20-unit padding matches the plain mode's.
