# src/xrGame/UITeamHeader.cpp

> The header strip above one team's scoreboard: authored column labels and authored aggregate fields refreshed from the team panel each frame.

**Needs** — [`UITeamHeader.h`](UITeamHeader.h.md) · [`UITeamState.h`](UITeamState.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`UITeamHeader.h`](UITeamHeader.h.md); callers name that, not this file.
**Tier floor** — T3: label layout plus one formatted line per aggregate.

## Purpose

Two authored node kinds live under a team header: a `column`, which is a static label
over a scoreboard column, and a `field`, which is a running aggregate about the team as
a whole. Only fields update; columns are laid out once and never touched again.

The split into its own file is arbitrary — it is forty lines of the team panel — but the
header is instantiated once per scroll panel while the team panel is one per team, so
keeping them apart keeps the ownership obvious.

## State

```text
RECORD TeamHeader EXTENDS Window
  parent          : TeamPanel            # the source of every field value
  column_labels   : map<text, Static>    # authored name -> widget; never updated
  field_widgets   : map<text, Static>    # authored name -> widget
  field_captions  : map<text, text>      # authored name -> localized caption,
                                         # resolved once at build time
```

**Invariant** — a caption is translated exactly once, when the field is created. Doing it
per frame would hit the string table every frame for text that cannot change without a
UI reset, which rebuilds the header anyway.

## `Init`

**Contract** — lays the header out from a named node, then walks that node's `column` and
`field` children in document order, creating one label per child keyed by its `name`
attribute. As with the scoreboard row, the document's traversal root is saved and
restored around the walk so the children are addressed relative to the header's node.

## `Update`

**Contract** — for every field, asks the owning team panel for the integer behind that
field's name and renders it as `caption: value`. Nothing here knows what any field means;
the team panel owns that table (see [`UITeamState.cpp`](UITeamState.cpp.md)).

```text
FUNCTION update(header)
  FOR EACH (name, widget) IN header.field_widgets
    value = header.parent.field_value(name)     # -1 when the name is unknown
    widget.text = header.field_captions[name] + ": " + value
```

**Notes** — an unknown field name renders as `-1` rather than failing, so a layout
document referring to a field this mode does not provide degrades to a visible wrong
number instead of a crash. That is worth preserving only if the rebuild also keeps the
shipped layout documents; otherwise treat an unknown name as an authoring error.
