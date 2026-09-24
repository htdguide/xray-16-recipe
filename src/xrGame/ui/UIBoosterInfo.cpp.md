# src/xrGame/ui/UIBoosterInfo.cpp

> "What will this do to me": the rows of a consumable's description, each one a raw
> configuration number turned into a fraction of what the level can do to you.

**Needs** — [`UIBoosterInfo.h`](UIBoosterInfo.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../../xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIBoosterInfo.h`](UIBoosterInfo.h.md)
**Tier floor** — T3: configuration reads plus vertical stacking

## Purpose

An item's description must say what taking it does. The effects are authored as raw numbers in
the item's configuration, in units that mean nothing to a player, so this panel normalises
them the same way the condition panel normalises protection — against the worst the current
level can do — and then stacks one labelled row per effect the item actually has.

## State

```text
RECORD BoosterPanel
  rows       : row [one per booster kind]   # all pre-built, most hidden at any time
  satiety    : row                          # nourishment, a separate configuration key
  anabiotic  : row                          # surge survival, a per-section special case
  duration   : row                          # how long the effect lasts
  separator  : widget                       # a rule drawn above the first row
```

Invariant: every row is built once at initialisation and **detached**, not hidden. `SetInfo`
detaches everything and re-attaches only the rows it wants, in order, positioning each below
the previous. Visibility is expressed as membership in the widget tree, which is also what
makes the vertical stacking trivial.

## The caption table

**Contract** — Thirteen caption identifiers, in the booster enumeration's own order, mapping
each booster kind to a localization key: health, power, radiation, bleeding, carry weight, and
then the eight protection and immunity kinds. The order is frozen by the enumeration, which is
shared with the condition simulation.

## `InitFromXml`

**Contract** — Build every row from the panel's element, one child element per booster named
by the *same* section-name table the simulation uses, plus the three specials. Returns false
when the panel element is absent.

**Invariants** — The rows' element names are the booster sections' own names, so adding a
booster kind to the simulation and to the layout is enough; no code changes. The document's
local root is reset to the panel element before each row, because each row's own
initialisation moves it.

## `SetInfo` — the normalisation

**Contract** — Show the rows for one item section. This is where the raw numbers become
readable.

```text
FUNCTION set_info(section)
  detach everything; re-attach the separator
  actor = the current view entity; RETURN IF not the player
  y = below the separator

  FOR EACH booster kind
    IF section's configuration does not mention it THEN CONTINUE
    value = configuration value; skip zero

    # The denominator depends on what KIND of effect it is.
    CASE the kind
      health / power / bleeding / carry weight  -> maximum = 1        # already a fraction
      radiation restoration                     -> maximum = -1       # sign flip: less is better
      burn immunity        -> maximum = the level's maximum fire power
      shock immunity       -> maximum = the level's maximum electrical power
      radiation immunity or protection  -> maximum = the level's maximum radiation power
      telepathic immunity or protection -> maximum = the level's maximum psi power
      chemical immunity or protection   -> maximum = the level's maximum acid power

    row.value = value / maximum
    place the row at y; y = y + row height; attach it

  IF the section declares nourishment THEN show that row
  IF the section IS the surge-survival drug THEN show that row unconditionally
  IF the section declares an effect duration THEN show that row

  panel height = y
```

**Invariants**

- **The level's maxima are the same denominators the condition panel uses**
  ([`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md)). Sharing them is what makes "this suit
  gives you 30%" and "this drug gives you 10%" comparable on the same scale. A rebuild that
  normalises the two panels differently makes the item description lie.
- **Radiation restoration divides by −1**, which is a sign flip dressed as a normalisation: a
  drug that *removes* radiation stores a negative number, and the panel must show it as a
  positive benefit.
- A zero-valued effect is skipped entirely rather than shown as zero.
- The panel's height is set from the accumulated stack, so the enclosing description can lay
  out around it.

**Notes** — The surge-survival row is triggered by **comparing the section name to a literal**.
That is the one hardcoded item name in the file and it is a defect: a rebuild should give the
effect a configuration key like every other. It is documented because the shipped data relies
on it.

## `UIBoosterInfoItem::Init`

**Contract** — One row: a caption and a value, plus four formatting attributes read from the
value element — a magnitude multiplier, whether to show an explicit sign, a unit suffix run
through the string table, and optionally a *negative-variant* caption icon. When a negative
icon is named, the positive one is taken from the caption's own texture attribute and both are
remembered.

## `UIBoosterInfoItem::SetValue`

**Contract** — Format and display one number.

```text
FUNCTION set_value(v)
  v = v * magnitude                       # 100 in the shipped data: a percentage
  text = show_sign ? signed integer : unsigned integer
  IF a unit suffix exists THEN text = text + " " + suffix
  display text in a fixed grey
  IF a negative caption icon exists
    caption icon = (v >= 0) ? the positive icon : the negative one
```

**Invariants** — The value is rounded to a whole number, so a magnitude that does not scale
the value into a readable integer range produces "0". The colour is fixed rather than derived
from the sign; the *icon* is what carries good-versus-bad, which is why the two-icon mechanism
exists at all.
