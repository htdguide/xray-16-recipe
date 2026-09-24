# src/xrGame/ui/UIRankingsCoC.cpp

> One row of the PDA ranking page: a row that asks a script every frame whether it should exist,
> and inserts or withdraws itself from the list accordingly.

**Needs** — [`UIRankingsCoC.h`](UIRankingsCoC.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIRankingsCoC.h`](UIRankingsCoC.h.md)
**Tier floor** — T3: pure screen logic over a script query and a layout document

## Purpose

The ranking page lists factions or characters whose visibility, text and icon are all decided by
game scripts rather than by the engine. Rather than have the page rebuild its list when something
changes, each row owns the decision: it holds an index, and every frame it asks a named script
function whether that index should be visible right now. A row that should be visible and is not
in the list inserts itself and refreshes all four of its fields at once; a row that should not be
visible and is in the list removes itself.

This inverts the usual "the container populates its children" arrangement, and it is the reason the
page needs no change notification from the script layer at all. The cost is one script call per row
per frame.

## State

```text
RECORD RankingRow
  parent       : ScrollContainer   # the list this row inserts itself into
  index        : int (8-bit)       # the argument every script query is passed
  name         : Widget            # coloured, newline-aware text
  description  : Widget            # coloured, newline-aware text; grows the row
  icon         : Widget
  hint         : HintWindow        # owned outright, NOT a child of this row
```

Invariants:

- `hint` is not attached to the widget tree, so it is neither drawn nor destroyed by the tree. It is
  drawn explicitly by `DrawHint` and released with the row.
- membership in `parent` and the row's own shown flag are always changed together; the pair
  "in the list but hidden" or "shown but not in the list" is never left standing.
- The row is created hidden and detached; it only ever becomes visible through `Update`.

## `CUIRankingsCoC`

**Contract** — Constructed against the scroll container it will later insert itself into. Owns its
hint window; everything else belongs to the widget tree.

## `init_from_xml`

**Contract** — Configures the row from a layout document, choosing between two named templates: one
for the player's own row and one for every other row. Reads the row frame, then descends into the
template's subtree so the four element names below are looked up relative to it, then restores the
document's previous root. Finishes hidden.

The two template names are frozen, because they are element names in shipped layout documents:
`coc_ranking_itm_actor` for the unique (player) row and `coc_ranking_itm` for the rest; within
either, the children are `name`, `descr`, `icon` and `hint_wnd`.

```text
FUNCTION init_from_xml(document, index, unique)
  template <- "coc_ranking_itm_actor" IF unique ELSE "coc_ranking_itm"
  configure this row as a window from document[template]
  saved_root <- document.local_root
  document.local_root <- node of document[template]
  self.index <- index
  name        <- static child "name"
  description <- static child "descr"
  icon        <- static child "icon"
  hint        <- hint window "hint_wnd"        # detached, owned
  document.local_root <- saved_root
  show(false)
```

**Notes** — The save/restore of the document's current root is how a nested element vocabulary is
read without every child path repeating the parent's path. It is a reader convenience, not a
decision a rebuild must copy, so long as child lookups are relative.

## `Update`

**Contract** — Runs each frame. Queries the script layer for this index's visibility. On a
visible-and-absent transition, refreshes all four fields from four further script queries, inserts
itself into the parent list and shows itself. On a not-visible-and-present transition, removes
itself from the list and hides. No script function present means no action at all — the row stays as
it was, which makes an incomplete script surface inert rather than fatal.

```text
FUNCTION Update()
  IF NOT script_has("pda.coc_rankings_can_show") THEN RETURN
  IF script_call_bool("pda.coc_rankings_can_show", index) THEN
    IF NOT in_parent_list() THEN
      IF script_has("pda.coc_rankings_set_name")        THEN SetName(...)
      IF script_has("pda.coc_rankings_set_description") THEN SetDescription(...)
      IF script_has("pda.coc_rankings_set_hint")        THEN SetHint(...)
      IF script_has("pda.coc_rankings_set_icon")        THEN SetIcon(...)
      parent.insert(self)          # without taking ownership
      show(true)
  ELSE
    IF in_parent_list() THEN
      parent.remove(self)
      show(false)
```

**Notes** — The five script entry point names (`pda.coc_rankings_can_show`,
`…_set_name`, `…_set_description`, `…_set_hint`, `…_set_icon`) are a frozen contract with shipped
Lua; see [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer).
The fields are refreshed only on the transition into the list, never while already in it, so a
script that changes a row's text without first hiding it will not be observed.

## `SetDescription`

**Contract** — Sets the description text with inline colour markup and newline escapes enabled,
grows the text widget to fit, then grows the *row* if the text plus a fixed bottom margin exceeds
the row's authored height. The row never shrinks below its authored height.

```text
FUNCTION SetDescription(text)
  description.coloring_mode  <- true
  description.newline_mode   <- true
  description.text <- text
  description.fit_height_to_text()
  needed <- description.height + 30       # bottom margin below the text
  IF needed > self.height THEN self.height <- needed
```

**Notes** — The 30-unit margin is in virtual-canvas units and is authored taste, not a constraint;
it exists so consecutive rows in the scroll list do not touch.

## `SetName`

**Contract** — Enables inline colour markup and newline escapes on the name widget, then sets the
text. The two mode flags are re-applied on every call rather than once at construction, which is
harmless and means the widget cannot be left in the wrong mode by anything else.

## `SetHint`

**Contract** — Sets the tooltip text, translating the argument through the localization string
table first — the script hands over an identifier, not a display string.

## `SetIcon`

**Contract** — Binds the icon's texture by registered icon name. An empty name is a no-op rather
than an error, so a script may decline to supply an icon.

## `DrawHint`

**Contract** — Draws the tooltip if and only if the cursor is inside this row's absolute rectangle.
Because the hint is not a child of the row, nothing else will draw it; the page calls this on each
row at a point in the frame after the list itself has been drawn, so tooltips sit above rows.

## `Reset`

**Contract** — Withdraws from the parent list and hides, then defers to the base reset. Called when
the page is closed or rebuilt, so that the next `Update` re-decides membership from scratch.

## `ParentHasMe` *(private)*

**Contract** — Linear search of the parent list for this row. Called up to twice per frame per row;
the lists are a few dozen entries at most, so the cost is not worth a membership flag that could
fall out of step with the list.
