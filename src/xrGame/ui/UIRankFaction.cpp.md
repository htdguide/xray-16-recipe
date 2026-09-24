# src/xrGame/ui/UIRankFaction.cpp

> One faction's standing: its strength, and a relationship meter built from four bars so that
> goodwill can run off both ends of its normal range.

**Needs** — [`UIRankFaction.h`](UIRankFaction.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`FactionState.h`](FactionState.h.md) · [`xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md)
**Used by** — [`UIRankFaction.h`](UIRankFaction.h.md)
**Tier floor** — T3.

## Purpose

The player's reputation with a faction is a signed number with no hard bound, and the page has
to show it on a fixed-width meter. The solution here is **four bars on one axis**: an inner
pair covering the ordinary range in each direction, and an outer pair that only begins to fill
once the inner one is full. That is the file's one idea.

## State

```text
RECORD FactionRow EXTENDS Window
  faction         : FactionState     # name, icon, home region, strength, goodwill
  sn, name, icon, icon_over          : Static    # place, name, emblem, emblem overlay
  location_static, location_value    : Static    # label and value
  power_static,    power_value       : Static    # label and value
  relation_minus, relation_center_minus,
  relation_center_plus, relation_plus : ProgressBar   # the meter, outer-inner-inner-outer
  origin_static, border_minus, border_plus,
  enemy_static, friend_static        : Static    # the meter's fixed decoration
  rating_up, rating_down             : Static    # the two movement arrows
  prev_sn         : int              # the place last shown, or none
```

**Invariants**

- The meter's four bars are all driven together and are mutually exclusive by sign: a positive
  goodwill zeroes both negative bars and vice versa. Nothing enforces it but the update, which
  always writes all four.
- The place shown is one-based and supplied by the list, not derived here; the row does not know
  how many factions there are.

## `init_from_xml`

**Contract** — apply the faction-row element and create all twenty widgets from it; every one is
required. Then shrink each of the two labels to its text and place its value ten units after it.
Finally initialise the movement arrows to neutral.

**Notes** — the two label-and-value pairs are aligned at construction rather than authored,
because the labels are localized and their widths differ per language. That pattern repeats
across the whole ranking page.

## `rating` — the movement arrows

**Contract** — compare the new place with the one last shown and tint the two arrows: **a
numerically larger place is worse**, so it lights the down arrow in red; a smaller place lights
the up arrow in green; an unchanged place leaves them alone. Forcing, or a first call, clears
both to neutral first.

```text
FUNCTION rating(new_place, force)
  IF force OR prev_place IS none THEN both arrows <- neutral
  IF prev_place < new_place THEN up <- neutral; down <- red;   prev_place <- new_place
  ELSE IF prev_place > new_place THEN up <- green; down <- neutral; prev_place <- new_place
  # equal: nothing changes, and the previous tint persists
```

**Notes** — the three colours are literals in this file: a pure green, a pure red, and an
off-white neutral. They are not read from the layout, which is inconsistent with the rest of the
chapter and means a style cannot restyle them.

The equal case deliberately leaves the previous tint standing, so an arrow lit by a change stays
lit until the place changes again. That is why the list can *force* a reset: when the standings
are reordered wholesale, every row's history is meaningless and all arrows are cleared.

## `update_info` — the four-bar meter

**Contract** — refresh the faction's state record, then write the place, the name, the emblem,
the home region, and the strength rounded to a whole number. Then drive the meter from the
player's goodwill.

```text
FUNCTION update_meter(goodwill)
  IF goodwill > 0 THEN
    both negative bars <- 0
    inner_plus <- goodwill                             # clamps itself at its own maximum
    outer_plus <- max(0, goodwill - inner_plus.maximum)
  ELSE IF goodwill < 0 THEN
    both positive bars <- 0
    inner_minus <- -goodwill
    outer_minus <- max(0, -goodwill - inner_minus.maximum)
  ELSE
    all four <- 0
```

**Notes** — the **inner bar's own maximum is the hand-off point**, so how far goodwill must go
before the outer bar starts is set in the layout document, not here. That is what makes the
meter's scale authorable per game.

The strength is displayed with no decimals, which is how the sort order can look wrong: two
factions half a point apart show the same number and appear mis-sorted.
