# src/xrGame/ui/ui_af_params.cpp

> The artefact tooltip's property list: fifteen possible rows, each scaled, signed and coloured by
> an authored rule, of which only the non-zero ones are stacked.

**Needs** — [`ui_af_params.h`](ui_af_params.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`../Actor.h`](../Actor.h.md) · [`../ActorCondition.h`](../ActorCondition.h.md) · [`../inventory_item.h`](../inventory_item.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`ui_af_params.h`](ui_af_params.h.md)
**Tier floor** — T3: table-driven configuration reading and vertical stacking

## Purpose

An artefact's tooltip lists what it does to the player: how fast it restores health or drains power,
how much it protects against each damage type, how much extra weight it lets the player carry. Every
one of those is a number in the artefact's configuration section, and every one needs the same four
decisions — what to call it, how to scale it, whether a negative value is good news, and what unit
to suffix.

Those four decisions are a **table**, not code. Two tables, in fact: one for the five restoration
rates and one for the nine displayable damage immunities. Everything else in the file is driven off
them, which is what keeps adding a property to a data change plus one table row.

## State

```text
RECORD PropertyRow extends Picture       # UIArtefactParamItem
  caption        : optional<Widget>      # when absent, the row's own text is the caption
  value          : Widget
  magnitude      : real                  # the configured number is multiplied by this
  sign_inverse   : bool                  # a negative value is the good one
  unit           : text                  # suffix, already localized
  positive_colour, negative_colour : int
  texture_plus, texture_minus : optional<text>   # the caption's icon swaps with the sign

RECORD ArtefactPanel extends Window      # CUIArtefactParams
  property_line    : optional<Widget>    # a separator; its presence also selects the palette
  condition_row    : optional<PropertyRow>
  restore_rows     : PropertyRow[5]      # indexed by restoration type
  immunity_rows    : PropertyRow[9]      # indexed by damage type
  weight_row       : optional<PropertyRow>
```

Invariants:

- Every row is allocated at build time and **detached**; `SetInfo` detaches everything and re-
  attaches only the rows with a non-zero value, in a fixed order, each below the last. So the panel
  shows a different number of rows per artefact with no rebuilding.
- Rows are not owned by the widget tree — they are marked to survive detachment — and are released
  with the panel. Letting the tree own them would destroy them on the first detach-all.
- The presence of the separator line is used as a **proxy for which game's data is installed**, and
  selects one of two colour palettes. A rebuild must keep the proxy or find another discriminator;
  nothing else in the layout distinguishes them.

## The two tables

**Contract** — Each row of each table is (identifier, configuration key, caption identifier,
magnitude, sign inversion, unit). The restoration table additionally names the *player's* own
configuration key for the same quantity, which the oldest game's data needs to normalise against.

```text
restoration:
  health restore     "health_restore_speed"     vs player "satiety_health_v"     x100   +   "%"
  satiety restore    "satiety_restore_speed"    vs player "satiety_v"            x100   +   "%"
  power restore      "power_restore_speed"      vs player "satiety_power_v"      x1     +   none
  bleeding restore   "bleeding_restore_speed"   vs player "wound_incarnation_v"  x-100  inv "%"
  radiation restore  "radiation_restore_speed"  vs player "radiation_v"          x1     inv none

immunity (all x100, not inverted, unit "%"):
  radiation, burn, chemical burn, telepathic, shock, strike, wound, explosion, fire wound
```

**Invariants** — The restoration table is asserted at build time to cover every restoration type, so
adding a type to the enumeration without adding a row is caught rather than silently dropped. The
immunity table is not asserted, and is deliberately shorter than the damage-type enumeration: three
damage types exist that an artefact cannot protect against and that no row describes.

Bleeding is the one entry that is both **negatively scaled and sign-inverted** — the configured
number is a bleeding *rate*, so a negative rate is a benefit, and the two flags together turn it
into a positive percentage shown in the good colour. Getting either flag alone gives the right
number in the wrong colour, or the wrong number in the right one.

## `InitFromXml`

**Contract** — Builds the panel and every row it could ever need. Reports absence when the layout
has no artefact-parameters element, which is how a tooltip omits the panel entirely.

```text
FUNCTION InitFromXml(document) -> bool
  IF document has no "af_params" THEN RETURN false
  configure self from it; descend into it
  property_line <- optional static "prop_line"
  palette <- the newer games' palette IF property_line EXISTS ELSE the older games' palette

  # each row is tried under two element names: its bare key, and "static_" + its key
  condition_row <- row("condition", localized "ui_inv_af_condition")
  FOR EACH entry IN the restoration table: restore_rows[entry.id]  <- row(entry)
  FOR EACH entry IN the immunity table:    immunity_rows[entry.id] <- row(entry)
  weight_row <- row("additional_weight", localized weight caption)
  restore the document root; RETURN true
```

**Notes** — Trying each row under two element names is how one reader serves the two layout
vocabularies, which prefix these elements differently. A row that matches neither is dropped, and
every use of it afterwards is guarded — so a layout may simply omit properties it does not want
shown.

## `Check`

**Contract** — An item section is an artefact this panel describes exactly when it declares a
player-properties key. One test, no exclusions — unlike the weapon panel's.

## `SetInfo`

**Contract** — Detaches every row, then re-attaches and stacks the ones with a non-zero value, in a
fixed order: condition, then the five restoration rates, then the nine immunities, then carry
capacity. Finally sizes the panel to the stack. Does nothing without a player character, since two
of the normalisations need the player's own configuration.

```text
FUNCTION SetInfo(item)
  detach everything; re-attach the separator line if there is one
  player <- the current view entity as a player character; IF none THEN RETURN

  artefact_section  <- item's configuration section
  condition_section <- the player's condition section, or the player's own section
  absorption_section<- the artefact's named absorption section
  h <- the separator's bottom edge, or 0

  # the stacking step, applied to every row that is shown
  place(row, value):
    row.set_value(value)
    row.y <- h
    h    <- h + row.height
    attach row

  IF condition_row EXISTS THEN place(condition_row, item.condition)

  old_data <- the installed material library is the oldest game's

  FOR EACH entry IN the restoration table
    IF no row THEN CONTINUE
    v <- configuration[artefact_section][entry.key]
    IF v is zero THEN CONTINUE                    # the property is simply absent
    v <- v * item.condition                       # a worn artefact does less
    IF old_data THEN v <- v / configuration[condition_section][entry.player_key]
    place(restore_rows[entry.id], v)

  immunities <- the absorption section's immunity set, loaded for this data generation
  FOR EACH entry IN the immunity table
    IF no row THEN CONTINUE
    v <- immunities[entry.id]
    IF v is zero THEN CONTINUE
    v <- v * item.condition
    IF NOT old_data THEN v <- v / the player's maximum protection for that damage type
    place(immunity_rows[entry.id], v)

  IF weight_row EXISTS THEN
    v <- configuration[artefact_section]["additional_inventory_weight"] * item.condition
    IF v is non-zero THEN place(weight_row, v)

  self.height <- h
```

**Invariants** — **The two normalisations are mirror images.** Restoration rates are normalised
against the player's own rates *only* in the oldest game's data; immunities are normalised against
the player's maximum protection *only* in the newer games'. The discriminator is the installed
material library's version, and getting it backwards makes every number in the tooltip wrong by the
player's own scale.

Every value is scaled by the artefact's condition first, so a worn artefact shows proportionally
reduced properties — which is the tooltip telling the player the artefact is used up.

A zero value means the property is absent and the row is not shown at all, rather than shown as
zero. That is what makes the panel's height meaningful.

## `UIArtefactParamItem::Init`

**Contract** — Builds one row from a named element, reporting absence so the caller can drop it.
Reads a caption — as a child element if the layout has one, otherwise as the row's own text — and
then the value label in one of two ways:

- if the layout declares a `value` child, configure it as a widget and take the magnitude, sign
  inversion, unit and the two colours from its attributes;
- otherwise, place the value label **immediately after the caption's measured text**, taking the
  caption's own text style, and fall back to the caller-supplied magnitude, sign and unit and to the
  registry's named `green` and `red`.

Then, in both cases, the element's own attributes override: `magnitude`, `sign_inverse`,
`unit_str` (translated), `positive_color` and `negative_color`. Finally, if the layout supplies a
`texture_minus`, the caption's own texture is captured as the plus texture — so the caption's icon
can swap with the sign.

**Notes** — The fallback path measures the caption's text and a two-space gap **in canvas units
converted to screen width**, which is why it converts both before adding them: the measurement is in
screen pixels and the placement is in canvas units. A rebuild that measures in canvas units skips
the conversion and must skip both.

Reading the attributes after the two branches means an attribute always wins over a caller-supplied
default, in both layouts. That ordering is the whole reason the caller's arguments are defaults
rather than settings.

## `UIArtefactParamItem::SetValue`

**Contract** — Scales the value, formats it **always signed** to no decimals, appends the unit if
there is one, and colours it by sign — inverted when the row says so. If the row has a sign-swapped
caption icon, swaps that too.

```text
FUNCTION SetValue(value)
  value <- value * magnitude
  text  <- value formatted with an explicit sign, zero decimals
  IF unit is non-empty THEN text <- text + " " + unit
  value_label.text <- text

  good <- value >= 0
  IF sign_inverse THEN good <- NOT good
  value_label.colour <- positive_colour IF good ELSE negative_colour
  IF a minus texture EXISTS AND there is a caption THEN
    caption.texture <- texture_plus IF good ELSE texture_minus
```

**Notes** — The explicit sign on every value, including positives, is deliberate: an artefact's
properties are always *changes* to the player, so `+15 %` and `-15 %` must be visually parallel.
Zero counts as good, which only matters for a value that rounds to zero from below — it shows as
`-0` in the good colour. That is the shipped behaviour.
