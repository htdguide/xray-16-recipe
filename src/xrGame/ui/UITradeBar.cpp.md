# src/xrGame/ui/UITradeBar.cpp

> One side of the trade screen's summary line: a price and a weight limit, right-aligned as a group
> by packing each label leftward from its neighbour.

**Needs** — [`UITradeBar.h`](UITradeBar.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md)
**Used by** — [`UITradeBar.h`](UITradeBar.h.md)
**Tier floor** — T3: text measurement and horizontal packing

## Purpose

The trade screen shows, for each side, what the current offer is worth and what the carrier's weight
limit is. The two numbers vary in width as they change, and they must stay visually attached to each
other and to the caption — so the line is not laid out by fixed positions but **packed right to
left**: each label is shrunk to its text, then placed just left of the one after it.

This is the same packing idea as [`CUIWeightBar`](UIWeightBar.cpp.md), and the two files could be
one. They are separate because the trade line has a currency and the weight line does not.

## State

```text
RECORD TradeLine extends Picture
  caption    : optional<Widget>   # omitted in the newest game's layout
  price      : optional<Widget>
  weight_max : optional<Widget>
```

Invariants:

- Packing runs only when both the price and the weight labels exist; with either missing the line
  keeps its authored positions.
- `weight_max` is the anchor: it never moves, and everything else is placed relative to it.

## `init_from_xml`

**Contract** — Configures the line from a named subtree and builds its three labels relative to it,
restoring the document root afterwards. The caption is built **only outside the newest game's data**
— that game's layout folds the caption into the price label's own text — and, when built, is
immediately shrunk to its text so packing has a real width to work with. Element names are frozen:
`trade_caption`, `trade_price`, `trade_weight_max`.

## `UpdateData`

**Contract** — Formats both numbers and re-packs the line. The price carries the localized currency
name, the weight carries the localized kilogram abbreviation and is shown to one decimal in
parentheses. Both labels are shrunk to their text before packing.

```text
FUNCTION UpdateData(price, weight)
  IF price label EXISTS THEN
    price.text <- decimal(price) + " " + localized currency name
    price.fit_width_to_text()
  IF weight label EXISTS THEN
    weight.text <- "(" + weight to one decimal + " " + localized("st_kg") + ")"

  IF both EXIST THEN
    price.x   <- weight.x - price.width - 5
    IF caption EXISTS THEN caption.x <- price.x - caption.width - 5
```

**Notes** — The 5-unit gaps are authored spacing. The weight label is not shrunk to its text, only
the price and the caption are — so the anchor keeps its authored width and the two mobile labels
pack against it. Parentheses around the weight and no unit suffix on the price are the shipped
presentation, and the currency name comes from the localization tables, not from configuration, so a
localization can change it.
