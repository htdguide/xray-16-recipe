# src/xrGame/ui/UIOutfitInfo.cpp

> The protection comparison panel: for each damage kind it shows what you have, what you would
> have, and what your belt adds — normalised so that two games with entirely different
> protection scales read the same.

**Needs** — [`UIOutfitInfo.h`](UIOutfitInfo.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`CustomOutfit.h`](../CustomOutfit.h.md) · [`ActorHelmet.h`](../ActorHelmet.h.md) · [`Actor.h`](../Actor.h.md) · [`ActorCondition.h`](../ActorCondition.h.md) · [`player_hud.h`](../player_hud.h.md) · [`xrUICore/ProgressBar/UIDoubleProgressBar.h`](../../xrUICore/ProgressBar/UIDoubleProgressBar.h.md)
**Used by** — [`UIOutfitInfo.h`](UIOutfitInfo.h.md)
**Tier floor** — T3.

## Purpose

Answers "is this suit better than mine", for nine damage kinds at once. Two things make it more
than a list of numbers: the comparison is a *two-valued bar* — current and prospective on one
track — and the underlying numbers mean different things in different games, so the panel
normalises before displaying.

## The damage-kind table

A fixed, ordered table naming, for each damage kind, the layout element that supplies its row
and the localization identifier for its label: burn, shock, chemical burn, radiation, psi,
impact, laceration, explosion, and bullet. The order is the display order.

```text
RECORD ProtectionRow EXTENDS Static          # the label is this widget's own text
  progress  : DoubleProgressBar              # element "…:progress_immunity"
  value     : Static                         # element "…:static_value"
  magnitude : real                           # what to multiply the 0..1 fraction by
```

**Invariants**

- **Every row is optional and the panel has no fixed height.** A layout that defines four rows
  gets a four-row panel; the stack and the panel's size are recomputed from whichever rows are
  *shown*, not from whichever exist.
- A row displays *either* a bar with a number beside it, *or* a number alone. Which one is
  decided per row by the document, and the two carry completely different formatting — see below.
- The scale factor defaults to one when there is a bar and to a hundred when there is not, so a
  bar reads a fraction and a bare number reads a percentage.

## `InitFromXml` — building the rows

**Contract** — apply the panel's own element, find the stack's starting position, and create one
row per damage kind, trying the modern element name and then the older `static_`-prefixed one.
A row whose element is absent under both names is not created.

The starting position comes from a fallback chain: below a property line if the document has
one, else below a caption if it has one, else from the explicit position of a scroll-view
element. Each corresponds to one of the three games' layouts.

## `CUIOutfitImmunity::InitFromXml` — one row, three shapes

**Contract** — configure the row from its element and set its label from the localization
identifier. Then decide its shape:

```text
IF the element defines a progress bar THEN
  show the bar
IF the element defines a value widget AND this is not the third game THEN
  configure the value widget from that element and show it
ELSE IF there is no bar THEN                       # the first game's layout
  make the value widget inherit the row's own text style,
  place it immediately after the label: row.left + the label's offset + the label's
                                        rendered width + one space width
  show it;  default the scale factor to 100
ELSE
  hide the value widget
scale factor <- the element's "magnitude" attribute, or the default just chosen
```

**Notes**

- **The first game's row has no bar and no separate value element**, so the panel measures the
  label's rendered width and places the number right after it, making one line of text out of
  two widgets. That measurement is done in *screen* units — the label's width is converted from
  canvas units first — which means the placement is computed against the physical width. That is
  inconsistent with every other placement in the chapter and looks like an error; it is
  preserved because the shipped first-game layout was tuned around it.
- The third game suppresses the separate value widget even when its document defines one,
  showing the bar alone. Which game is running is read from a global, so a rebuild needs the
  same per-game switch.

## `SetProgressValue` — the two display forms

**Contract** — scale the current and comparison values by the row's factor, then either drive
the bar or write a coloured signed number.

```text
FUNCTION set_value(current, comparison, bonus)
  current <- current * magnitude;  comparison <- comparison * magnitude
  IF the bar is shown THEN
    IF current equals comparison AND there is a bonus THEN comparison <- bonus
    bar.set_two_positions(current, comparison)
    text <- rounded current, and "+ <bonus>" when there is a bonus
  ELSE
    text <- (current > 0 ? green : red) + signed current as a percentage
            and, with a bonus, the same again for the bonus
  value.text <- text
```

**Notes**

- **The bar's second value is overloaded.** It normally means "what you would have if you wore
  the other item"; when there is nothing to compare against but the belt contributes, the same
  track shows the belt's bonus instead. One bar, two meanings, distinguished only by whether the
  two protection values happen to be equal — which they also are when the two items genuinely
  match. A rebuild should give the bonus its own track or its own colour.
- The bare-number form uses the localization markup's inline colour names to tint the sign,
  which is why it is a string rather than a widget colour: the number and the bonus can differ
  in sign within one line.

## `UpdateInfo` (armour)

**Contract** — with no actor or no armour, hide every row when asked to hide empty rows, and
return. Otherwise, for each row: take the armour's protection for that damage kind, the other
armour's if one was given (otherwise the same value, so the bar shows no change), and the belt's
artefact contribution if asked. In the two later games, divide all of these by the *maximum
power* the actor's condition system assigns that damage kind, turning an absolute figure into a
fraction. Hide the row when both values are zero and hiding was requested. Finally restack.

The bullet-protection row is special in the two later games: it is not a flat figure but the
armour's per-bone armour value at the spine, scaled by the item's condition and normalised
against a separate maximum. It is computed after the loop and skipped inside it.

**Notes**

- **The normalisation is the reason this panel works across games.** The first game stores
  protection as a fraction already; the later two store it in absolute units whose scale is set
  by the condition system. Dividing by that maximum makes both read as zero-to-one, and the
  row's scale factor then turns it into whatever the layout wants to show. A rebuild that skips
  the division shows meaningless numbers on two of the three games.
- Which convention is in force is detected from the *armour itself* — whether its bone-protection
  type differs from the shared default — not from a game flag. That is a proxy, and an armour
  authored unusually would be read wrongly.
- The spine bone is looked up by a fixed name on the actor's skeleton. The comparison item's
  lookup recomputes the same bone from the same skeleton, which is redundant but harmless.

## `UpdateInfo` (helmet)

**Contract** — the same, always using the later games' normalisation, always comparing against
the same value when no second helmet is given, never adding a belt bonus, and taking bullet
protection from the *head* bone rather than the spine. Rows are never hidden.

**Notes** — the helmet form does not honour the hide-when-zero behaviour at all, so a helmet
panel shows every row it has. That asymmetry is not explained anywhere and looks unintended.

## `AdjustElements`

**Contract** — stack the visible rows from the starting position, each below the last at its own
height, and set the panel's height to the bottom of the stack while keeping its width.
