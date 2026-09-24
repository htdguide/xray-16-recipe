# src/xrGame/ui/UIInvUpgradeInfo.cpp

> The description panel beside the upgrade tree: name, cost, description, why-you-cannot, and the property list — laid out by stacking measured heights, so the panel's own height is whatever the content came to.

**Needs** — [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md) · [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`inventory_upgrade.h`](../inventory_upgrade.h.md) · [`Actor.h`](../Actor.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md)
**Tier floor** — T3.

## Purpose

What the player reads when hovering an upgrade. Two decisions carry the file: the panel is
**laid out by measurement**, not by authored positions, because every field's height depends
on its text; and the *refusal message* is composed from a verdict by a branch table that
**differs between two of the games**.

## State

```text
RECORD UpgradeInfoPanel EXTENDS Window
  upgrade    : optional<Upgrade>    # none means the panel draws nothing at all
  background : FrameWindow          # resized with the panel
  name, cost, description, prerequisites : Label
  properties : PropertyList
```

**Invariants**

- **Nothing draws while no upgrade is set.** Not hidden — the draw is skipped — so the panel
  can be attached permanently and simply disappear.
- Setting the *same* upgrade again is a no-op and returns false. Every hover would otherwise
  recompose and re-measure the whole panel each frame.
- The cost field is optional: one game's upgrade bench does not show a price.

## Composing the refusal

**Contract** — when the upgrade is *known* to the player, the prerequisite field states
either that it is installed or why it cannot be:

```text
FUNCTION compose_status(verdict)
  colour := red by default                      # except in the second game, which
                                                # does not colour this field
  CASE verdict OF
    already installed -> colour := green; text := localize("installed")
    not yet learned   -> text := header + " - " + localize("unknown"); hide the cost
    exclusive sibling -> text := header + " - " + localize("group")
    cannot do         -> text := header + " - " + localize("cant do")
    installable, unaffordable, or story-blocked
                      -> in the second game: the upgrade's own prerequisite text
                         otherwise: nothing, unless story-blocked, which falls through
    missing prerequisite (and the fall-through above)
                      -> header + ":" + the upgrade's prerequisite text,
                         and for a missing prerequisite, a further " - parents" line
  # `header` is the localized "cannot upgrade because"
```

**Notes** — every string is a localization identifier and the line separators are the
**two-character newline escape** chapter 15 describes, embedded in the composed string and
resolved by the text engine, not by the composer. That is why the composition can be plain
concatenation.

The second game is special-cased in three places — no colour, a different escape convention,
and a different set of verdicts that produce prerequisite text. Its upgrade data was authored
against a different presentation and the engine reproduces both. A rebuild that unifies them
will show the wrong message in one of the two games.

The "cannot afford" verdict deliberately produces **no** message in the third game: the cost
field is already showing the price, and the player can see they cannot pay it.

## Layout by measurement

**Contract** — after the text is set, every field is fitted to its content and then stacked:

```text
FUNCTION reflow()
  fit each of name, cost, description, prerequisites to its own text height
  y := name.bottom + 5
  IF a cost field exists THEN place cost at y; y := cost.bottom + 5
  place description at y;    y := description.bottom + 5
  place prerequisites at y;  y := prerequisites.bottom + 5
  place the property list at y
  panel.height := property_list.bottom + 10
  background.height := panel.height
```

Only the vertical positions are recomputed; each field keeps its authored horizontal
position, so columns stay aligned.

**Notes** — the panel and its background frame are resized together every time. That is what
makes the frame hug the text instead of being a fixed box with a ragged gap, and it is the
reason the background is a nine-slice frame rather than a picture — see chapter 15 on why a
nine-slice is how a panel scales to an arbitrary size from a small texture.

The five-unit gaps and the ten-unit bottom margin are literal and uniform. No reason beyond
the look.

## The cost

**Contract** — the price is a **script call** taking the upgrade's configuration section and
returning a display string. A missing script function is fatal.

**Notes** — the price is a formatted string, not a number, because the upgrade economy is a
mod's to define: the function may return a currency amount, a barter requirement, or a word.
The engine does not know what an upgrade costs and deliberately does not.

## Unknown upgrades

**Contract** — an upgrade the player has not learned about shows only its name and
description; the prerequisite field, the property list and the cost are hidden.

**Notes** — name and description are still shown, which reads as a leak and is not: an
unlearned upgrade's description in the shipped data is a placeholder, and the tree node
beside it is already drawn in the unknown state.
