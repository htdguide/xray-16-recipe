# src/xrGame/ui/UIMapLegend.cpp

> The map legend: a closable panel whose rows are read wholesale from the layout document, so
> that what the markers mean is authored data rather than code.

**Needs** — [`UIMapLegend.h`](UIMapLegend.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md)
**Used by** — [`UIMapLegend.h`](UIMapLegend.h.md)
**Tier floor** — T3.

## Purpose

Explains the map's marker vocabulary to the player. The whole content — how many rows, which
images, which captions — comes from a repeated element in the map screen's layout document, so
adding a marker kind is a data change. This file contributes only the frame, the close
behaviour and the per-row height rule.

## State

```text
RECORD Legend EXTENDS Window
  background : FrameWindow   # element "background_frame"
  caption    : Static        # element "t_caption"
  btn_close  : Button        # element "btn_close"
  list       : ScrollView    # element "legend_list"; holds one row per "item" element

RECORD LegendRow EXTENDS Window
  images : list<Static>      # elements "image" and optionally "image_1".."image_3"
  text   : Static            # element "text_static"
```

**Invariants**

- A row's height is the greater of its authored height and the bottom of its wrapped text.
  Rows therefore grow downward for long captions and never shrink below the authored height
  that the images need. This is the only layout decision in the file.
- Exactly one image is required per row; the other three are optional. A marker whose meaning
  depends on a set of icons — say four difficulty grades — is one row with four images.

## `init_from_xml` (the panel)

**Contract** — apply the named section, then build the frame, the caption, the close button
and the scroll view from elements *inside* that section, then build one row per repeated
`item` element under the list's own element and adopt each into the scroll view. Restores the
document's previous local root on the way out.

**Notes** — the document's "current subtree" is moved three times and restored once, because
the same element names (`image`, `text_static`) repeat under every row and are only
unambiguous relative to that row. A rebuild whose layout reader takes full paths needs none of
this; the *decision* being preserved is that row elements are addressed relative to their row.

## `init_from_xml` (a row)

**Contract** — apply the indexed `item` element to the row, create the mandatory image and
whichever of the three optional ones the element defines, create the caption and fit its
height to its wrapped text, then set the row's height as described in the invariants.

## `SendMessage`

**Contract** — the close button hides the panel on press. Note that it acts on the *press*
notification, not on the click: the legend closes as the button goes down, without waiting for
release.
