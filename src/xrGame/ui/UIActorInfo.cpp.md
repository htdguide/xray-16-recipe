# src/xrGame/ui/UIActorInfo.cpp

> The player's statistics page: categories on the left, entries on the right, and one category
> — reputation — that is not a statistic at all.

**Needs** — [`UIActorInfo.h`](UIActorInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`../../xrUICore/Windows/UIFrameLineWnd.h`](../../xrUICore/Windows/UIFrameLineWnd.h.md) · [`../../xrUICore/Static/UIAnimatedStatic.h`](../../xrUICore/Static/UIAnimatedStatic.h.md) · [`../../xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md)
**Used by** — [`UIActorInfo.h`](UIActorInfo.h.md)
**Tier floor** — T3: list building from a registry

## Purpose

A master-detail page over the player's own statistics registry, plus a character panel. It is
one of the personal terminal's sections and is built the way every such section is: one layout
document, all widgets found by name.

The page's one structural decision is that **the master list is built from the layout
document, not from the registry**. The categories that exist are the ones the document lists,
in the document's order, and each carries an identifier that is then looked up in the
registry. A category with no registry entry shows blank rather than being omitted.

## State

See [`UIActorInfo.h`](UIActorInfo.h.md). The page owns no data; both lists are rebuilt from
scratch on every show and every selection change.

## `Init`

**Contract** — Build the page from `actor_statistic.xml`. Returns false when that document is
absent, and the personal terminal then omits this section entirely. Otherwise builds, in
order: two frame windows (character side and info side), a frame line header for each, an
animated icon in the character header, the two scrolling lists, a character panel from a
second document, and finally two groups of decorative statics — one per side — through the
layout reader's automatic-static mechanism.

**Notes** — The character panel is initialised with the *enclosing window's* position and
size rather than its own, which is the chapter's idiom for "fill this frame".

## `Show`

**Contract** — On being shown: re-read the player's character record into the panel, set the
character header's title to the player's name, and refill the master list. Nothing happens on
being hidden.

**Invariants** — Refreshing on show, rather than on change, is the whole update policy of this
page. Statistics change while the page is closed and are simply re-read next time.

## `FillPointsInfo`

**Contract** — Rebuild the master list. Re-loads the layout document — a second time, this one
not cached in the page — and creates one row per `master_part` element, in document order.

For each row, the identifier comes from the element and the *value* comes from one of three
places:

```text
FOR EACH master_part element, in order
  row = new header row, configured from that element
  id  = the element's id attribute

  IF id = "foo"            -> leave both fields as authored   # a spacer row
  ELSE IF id = "reputation"
      # Not a statistic. Reputation is a character attribute, shown as a
      # localized band name coloured by the band.
      row.value = reputation_as_text(actor.reputation)
      row.value colour = reputation_colour(actor.reputation)
  ELSE
      points = actor.statistics.points_for(id)
      row.value = (points = -1) ? "" : points     # -1 means "this category has no total"
  add the row

select the SECOND row
```

**Invariants**

- `"foo"` is a sentinel identifier meaning "this row is decoration"; it is also used in the
  alternative construction path below. It is frozen by the shipped document.
- `-1` from the statistics registry means *no total*, which is different from a total of zero,
  and is shown as an empty field.
- **Selecting the second row** on every refresh is what makes the detail list non-empty when
  the page opens. The first row is the spacer; the second is the first real category. The
  index is hardcoded and depends on the document's layout.

The reputation case is the interesting one: the master list is nominally a statistics view,
and one of its rows is not a statistic. That is presentation convenience — reputation belongs
on this page and there was nowhere else to put it — and it is why the detail path also
special-cases it.

## `FillMasterPart`

**Contract** — The same row construction for an alternative build of the game, where the
master rows are named by *path* (`master_part_<key>`) rather than enumerated, and the key set
comes from the registry rather than the document. Same value rules, including the reputation
case and the `-1` sentinel.

**Notes** — Only one of the two paths is compiled. The recipe keeps both because they are
genuinely different policies — document-driven versus registry-driven — and which one a
rebuild wants depends on whether it wants modders to be able to add categories from data.

## `FillPointsDetail`

**Contract** — Rebuild the detail list for one category identifier.

```text
FUNCTION fill_points_detail(id)
  clear the detail list
  re-load the layout document; scope it to the statistics element

  path = "detail_part_" + id
  IF that element does not exist THEN path = "detail_part_def"   # a generic row shape

  header title = localize("st_detail_list_for_" + id)

  IF id = "reputation" THEN fill_reputation_details(); RETURN

  FOR EACH entry IN actor.statistics.section(id), in registry order
    row = new detail row from path
    row.index = running counter, followed by a period
    row.name  = localize(entry.key), height fitted to the wrapped text

    IF entry has no string value
      row.value  = "x" + entry.count
      row.points = entry.points
    ELSE
      row.value  = entry's string value
      row.points = ""                      # a string entry has no point total

    row height = max(authored height, name field's bottom + 3)
    add the row
```

**Invariants** — A row's height is grown to fit its wrapped name, never shrunk below the
authored height. The 3 is padding below the text and is the only tunable in the layout that
is in code rather than in data.

The per-category element with a generic fallback is how one document describes a dozen
differently shaped detail rows without the code knowing any of them.

## `FillReputationDetails`

**Contract** — The reputation detail is not a statistics section: it is one row per community,
listing how that community feels about the player.

```text
FUNCTION fill_reputation_details()
  communities = the list of community names in the document
  neutral_offset = reputation_relation(actor.reputation_band, neutral_band)

  FOR EACH community
    row = new detail row
    row.name = the community's identifier

    goodwill = relation_registry.community_goodwill(community, actor)
             + community_relation(actor.community, community)
             + neutral_offset

    row.value = goodwill_as_text(goodwill)       # a localized band name
    row.value colour = goodwill_colour(goodwill)
    row.points = goodwill                        # the raw number
```

**Invariants** — The three-term sum is the decision. A community's attitude to the player is
*not* a single stored number: it is the registry's recorded goodwill, plus the standing
relation between the player's own community and that one, plus an offset derived from the
player's reputation band relative to neutral. All three must be included or the screen
disagrees with how the game actually treats the player.

The community list comes from the document, not from the game's community table, so a
community the document omits is simply not shown.

## `CUIActorStaticticHeader`

**Contract** — A master row. Clicking it, with the primary button and unless its identifier is
`"total"`, asks the list to select it; selecting it raises its first field's alpha to full and
**calls back into the page to fill the detail list**. Deselecting restores the alpha the
document authored, which is remembered per row at construction — the rows do not share a
colour.

`"total"` is a second sentinel: a summary row that may not be drilled into.

## `CUIActorStaticticDetail`

**Contract** — A detail row: four text fields configured from the element, plus the automatic
statics. No behaviour.
