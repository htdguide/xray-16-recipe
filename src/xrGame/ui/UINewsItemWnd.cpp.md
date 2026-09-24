# src/xrGame/ui/UINewsItemWnd.cpp

> One row of the PDA news list: it lays the timestamp, caption and body out on one flow and
> grows the row to fit the taller of the text and the icon.

**Needs** — [`UINewsItemWnd.h`](UINewsItemWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`game_news.h`](../game_news.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UINewsItemWnd.h`](UINewsItemWnd.h.md)
**Tier floor** — T3.

## Purpose

A row of authored widgets and about ten lines of layout arithmetic. Its one real job is to work
against three games' layout documents, which name and arrange the same four widgets differently
and, in the first game's case, do not have a caption at all.

## State

```text
RECORD NewsRow EXTENDS Window
  image   : Static            # required
  caption : optional<Static>  # absent in the first game's layout
  text    : optional<Static>  # element "text_static", or "text_cont"
  date    : optional<Static>  # element "date_static", or "date_text_cont"
```

## `Init`

**Contract** — apply the named subtree to the row, then create the four widgets relative to it,
each by its preferred element name and falling back to the older name. The icon is required; the
rest are optional.

**Notes** — the fallback names are not aliases with the same meaning: the older document nests
the text in a container element, which is why the name is different. A rebuild with one layout
keeps only the first name and drops the caption's optionality by supplying one.

## `Setup`

**Contract** — render the record's receipt time as a date-and-time string, append a separator
when there is a caption, and shrink the date widget to its text. Place the caption five units
after the date and cap its width so that date plus caption fits the body's width. Set the body
text and grow it to its wrapped height. Bind the icon. Set the row's height to the greater of the
body's bottom plus six units and the icon's bottom.

```text
FUNCTION setup(news)
  date.text <- clock_and_date(news.received_at) + (caption EXISTS ? " -" : "")
  date.shrink_to_text()
  IF caption EXISTS THEN
    caption.position.x <- date.right + 5
    caption.text  <- localize(news.caption)
    caption.width <- min(text.width - date.width - 5, caption.width)
  text.text <- localize(news.text);  text.fit_height_to_text()
  image.bind(news.texture)
  height <- max(text.bottom + 6, image.bottom)
```

**Notes**

- **The date and caption form one line whose total width is the body's width.** The caption is
  clamped rather than wrapped, so a long caption is truncated by its widget's own ellipsis rule
  rather than pushing the row taller. That keeps the list's rows visually regular.
- The trailing separator is added only when there is a caption to separate from, which is the
  whole of the first game's special case at this level.
- The two padding constants — five units between date and caption, six below the body — are
  visual spacing with no further meaning.
