# src/xrGame/ui/UIRankingWnd.cpp

> The PDA's ranking page: five optional panels over one timed refresh, most of whose content is
> supplied by named script functions rather than by the engine.

**Needs** — [`UIRankingWnd.h`](UIRankingWnd.h.md) · [`UIRankFaction.h`](UIRankFaction.h.md) · [`UIAchievements.h`](UIAchievements.h.md) · [`UIRankingsCoC.h`](UIRankingsCoC.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`Actor.h`](../Actor.h.md) · [`relation_registry.h`](../relation_registry.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIRankingWnd.h`](UIRankingWnd.h.md)
**Tier floor** — T3.

## Purpose

The page that tells the player who they have become. It is the chapter's clearest example of a
screen that is **mostly a rendering surface for the script layer**: the statistics lines, the
best-kill trophy, the favourite weapon, the achievement conditions and the leaderboard's size
and contents all come from named Lua functions, and the engine supplies only the layout, the
refresh timer and the faction standings.

Everything on it is optional. Every panel is created only if the layout document defines it and
guarded at every use, which is how one implementation serves three games with three completely
different ranking pages — and one game with none at all.

## State

```text
RECORD RankingPage EXTENDS Window
  actor_info       : optional<CharacterInfo>   # portrait, name, community, reputation
  money_value      : optional<Static>
  stat_info        : list<Static>              # one value widget per "stat" element
  factions_list    : optional<ScrollView>      # rows of FactionRow, sorted by strength
  achievements     : optional<ScrollView>
  achieves_vec     : list<Achievement>
  coc_ranking_vec  : list<LeaderboardRow>      # the top N
  coc_ranking_actor: optional<LeaderboardRow>  # the player's own line, outside the list
  monster_icon_back, monster_icon,
  favorite_weapon_icon : optional<Static>
  delay            : int                       # refresh interval, from the document
  previous_time    : int
  last_*_icon      : text                      # what each trophy icon currently shows
```

**Invariants**

- **The page refreshes on a timer, not per frame** — every three seconds unless the document
  says otherwise — because refreshing means calling half a dozen script functions and rebuilding
  the faction standings.
- The three trophy icons are guarded by a remembered name each, so an unchanged answer from the
  script costs nothing. Resetting the page clears those names *and* turns the textures off, so
  the next refresh re-binds.
- A faction row's *place* is its index in the sorted list plus one. The row does not compute it.

## `Init`

**Contract** — load the ranking document, declining if absent. Apply the page element and read
the refresh interval. Then build, each optionally: the backgrounds; the player's character
panel, with its community icon nudged to sit after its caption; the money label and value,
aligned the same way; a centre caption whose authored text is *concatenated* with a localized
string; the faction decoration; one caption-and-value pair per repeated statistics element; the
faction list; the two trophy panels; the achievements list; and the leaderboard.

The faction list is filled from a configuration section naming the factions to show, one row
each. The achievements list likewise, each row configured from its own configuration section:
name, description, hint, icon, the *script function* that decides whether it is earned, and
whether it can be earned more than once. The leaderboard's length is asked of a script function,
defaulting to fifty.

```text
FUNCTION init() -> bool
  doc <- load_layout("pda_ranking.xml");  IF absent THEN RETURN false
  apply(doc, "main_wnd", self);  delay <- doc.attribute("main_wnd", "delay") OR 3000
  ... build the optional decoration ...
  FOR i, element IN repeated "stat" elements under "stat_info"
    caption <- static from ("stat", i);  caption.shrink_to_text()
    value   <- static from ("stat", i)          # the SAME element, twice
    value.colour <- the document's "value" colour
    value.position <- just right of caption, 5 units
    stat_info.append(value)
  IF the document has a faction list THEN
    it sorts by faction strength, descending
    FOR EACH faction named in configuration section "pda_rank_communities": add a row
  ... the trophy panels ...
  IF the document has an achievements list THEN
    FOR EACH id in configuration section "achievements": add a row from section <id>
  count <- script "pda.get_rankings_array_size"() OR 50
  IF the document has a leaderboard THEN add `count` rows, numbered from one
  IF the document has an actor leaderboard line THEN add one row numbered count + 1
  RETURN true
```

**Notes**

- **A statistics line is one element used twice.** The caption and the value are both built from
  the same layout element at the same index, so they start identical; the caption is then shrunk
  to its authored text and the value is moved to sit after it and given the shared value colour.
  This means a statistics line needs only *one* element in the document, with the caption as its
  text, and the value's geometry is derived. A rebuild will find this confusing and should
  author two elements; the behaviour to preserve is that the value follows the caption's
  *rendered* width, so it stays aligned in every language.
- The centre caption's authored text is a *prefix* to which a localized string is appended.
  Nothing else in the chapter does this; it exists so the document can supply a symbol or number
  in front of a translated phrase.
- The achievement rows are the only part of the page whose *definition* is configuration rather
  than layout: five strings and a flag per achievement, one of which names the script function
  that evaluates it. Adding an achievement is a data change.
- An achievement whose configuration section is missing is skipped with a log line rather than
  being fatal, because the configuration section listing them is often edited by hand.

## `update_info` — the refresh

**Contract** — update every achievement and leaderboard row; re-read the statistics, the
best-kill trophy and the favourite weapon; then, for the faction list: decide whether the
standings have been reordered by checking whether any row's remembered place still matches its
current index, and refresh every row with its place, forcing the arrow reset when they have.
Finally re-sort the list.

**Notes** — the reorder check is what makes the movement arrows meaningful. Without it, a
reorder would make every row's arrow point in whatever direction its index happened to move,
which is noise; with it, either every arrow is meaningful or all are cleared.

## The three script-fed panels

**Contract** —

- **Statistics** — the first line is the engine's own: the elapsed game time formatted as a
  period. The remaining lines come from a script function called with the line's index. Every
  line is tinted a fixed grey and rendered with inline markup enabled, so a script can colour
  its own values.
- **Best monster** — two script functions supply a backdrop and an icon name; each is bound only
  when it differs from what is already shown, and an empty answer abandons the refresh.
- **Favourite weapon** — a script function supplies a weapon's section name; the icon is then
  cut from the *upgrade* icon atlas using four coordinates from that section's configuration and
  drawn at eighty per cent scale, width-corrected for the canvas aspect.

**Notes**

- Whether the engine's own elapsed-time line occupies the first slot is decided by a **proxy
  test** — whether the page also has a character panel and a money readout — which the source
  labels as a hack for telling the games apart. A rebuild should declare the line in the layout
  like any other.
- The favourite-weapon icon reads from the *upgrade* atlas rather than the inventory atlas,
  because that atlas carries large renders. One weapon is special-cased onto a different atlas
  entirely, with no discoverable reason beyond that its render was placed there.
- The eighty-per-cent scale and the grey are literals.

## `Show`, `Update`, `DrawHint`, `ResetAll`

**Contract** — showing re-binds the character panel to the actor, writes the money with the
localized currency name, refreshes everything and updates once synchronously so the page is
correct on its first frame. Updating refreshes when the interval has elapsed and otherwise only
pumps the child tree, and only while shown. Hints are drawn out of band for every achievement
and leaderboard row that is visible. Resetting clears the three remembered icon names, turns
those textures off, and resets every achievement and leaderboard row.

**Notes** — hints are drawn separately from the tree because a hint must appear above the scroll
view that clips its owner. That is the same out-of-band pattern as the map screen's hint.

## `SortingLessFunction`

**Contract** — faction rows sort by strength, descending. The list applies it on every forced
refresh, so the standings reorder as the world changes.
