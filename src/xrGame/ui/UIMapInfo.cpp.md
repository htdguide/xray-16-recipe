# src/xrGame/ui/UIMapInfo.cpp

> The multiplayer map description panel: it looks for a map's own description file and, if one
> exists, builds a scrolling list of labelled, coloured, wrapped paragraphs from it.

**Needs** — [`UIMapInfo.h`](UIMapInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIMapInfo.h`](UIMapInfo.h.md)
**Tier floor** — T3.

## Purpose

Shows what a map is before it is chosen. The content is *per map data*, not per screen: each
map may ship a small description file, and the panel is a renderer for that file. A map
without one gets its name and nothing else, which is the expected case for user-installed
maps.

## State

```text
RECORD MapInfoPanel EXTENDS Window
  view       : ScrollView   # the only child; holds one paragraph widget per line of content
  large_desc : text         # the long description, harvested as a side effect of a rebuild
```

**Invariants** — the panel's own rectangle and the scroll view's size are set together and the
scroll view is left at the panel's origin, so the two are the same box. The scroll bar is the
content-sized kind.

## `InitMapInfo`

**Contract** — set the panel's position and size, give the scroll view the same size, bring it
up and put it on a content-sized scroll bar.

## `InitMap`

**Contract** — clear the panel and, for a named map, look for `text/map_desc/<map>.ltx` under
the configuration root. When it exists, emit: the map's localized name with its version in
square brackets; the player-count line; the supported game modes; and the short description.
When it does not, emit only the name. Also harvest the long description into the panel's own
field. A null map name clears and returns.

```text
FUNCTION init_map(map_name, map_ver)
  view.clear()
  IF map_name IS none THEN RETURN
  style <- load_layout("ui_mapinfo.xml")            # fonts and colours only

  file <- "text/map_desc/" + map_name + ".ltx"
  IF NOT config_root.exists(file) THEN
    view.add(paragraph styled by style's "map_name", text: localize(map_name))
    RETURN

  desc <- read_config(file)
  title <- localize(map_name) + (map_ver ? "[" + map_ver + "]" : "")
  view.add(paragraph styled by style's "map_name", text: title)

  header_colour, font <- style's "header" font definition
  body_colour         <- style's "txt:text" colour, black if absent

  emit_labelled("mp_players",     desc["map_info"]["players"]   OR "Unknown")
  emit_modes(desc["map_info"]["modes"])
  emit_labelled("mp_description", desc["map_info"]["short_desc"] OR "")
  large_desc <- localize(desc["map_info"]["large_desc"]) IF present

FUNCTION emit_labelled(label_id, value_id)
  # the label in the header colour, the value in the body colour, one trailing line break
  text <- localize(label_id) + ": " + colour_markup(body_colour)
                             + localize(value_id) + end_colour_markup + newline_escape
  p <- paragraph(font, header_colour, text) WITH inline markup enabled
  p.width <- view.desired_child_width;  p.fit_height_to_text()
  view.add(p, owned: true)
```

**Notes**

- **The paragraph is two colours in one widget.** The widget's own colour is the header
  colour; the value is switched to the body colour by an inline colour marker inside the
  string and switched back by a marker meaning *default*. That markup is part of the frozen
  localization string format, so composing it here is cheaper than two widgets and — more
  importantly — keeps the label and the value on one wrapped flow.
- **Every value is a localization identifier, including the ones read from the map's file.**
  The description file names strings; it does not contain them. That is what lets one map file
  serve every shipped language.
- **The game-mode line is not a translated list, it is a membership test.** The file's mode
  entry is a single string, and the panel checks it for three known substrings and joins
  whichever it finds with commas. Consequences: the order is fixed by the code, not by the
  file; a mode the code does not know about is invisible; and — because one mode's identifier
  is a substring of another's — a file naming only the team variant also matches the plain
  one, so the line reads as both. A rebuild that splits the entry on a separator fixes the
  last of these and changes what the shipped files display.
- Paragraph height is derived from the wrapped text after the width is set, and the widths come
  from the scroll view, so the panel needs no layout of its own.
