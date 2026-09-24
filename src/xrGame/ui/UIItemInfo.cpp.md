# src/xrGame/ui/UIItemInfo.cpp

> The item tooltip and description panel: a name, a weight, a price, an icon, and a scroll view into which each interested sub-panel appends itself only if the item is of its kind — so one widget describes a rifle, a suit, a sausage and an artefact.

**Needs** — [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIWpnParams.h`](UIWpnParams.h.md) · [`ui_af_params.h`](ui_af_params.h.md) · [`UIOutfitInfo.h`](UIOutfitInfo.h.md) · [`UIBoosterInfo.h`](UIBoosterInfo.h.md) · [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`inventory_item.h`](../inventory_item.h.md) · [`Weapon.h`](../Weapon.h.md) · [`CustomOutfit.h`](../CustomOutfit.h.md) · [`ActorHelmet.h`](../ActorHelmet.h.md) · [`eatable_item.h`](../eatable_item.h.md)
**Used by** — [`UIItemInfo.h`](UIItemInfo.h.md)
**Tier floor** — T3.

## Purpose

Everything the player can learn about an item without using it. It appears as a tooltip beside
the inventory grid, as a panel on the trade screen, and beside the upgrade bench — one class,
three placements, and the differences are all authored.

Its structure is **composition by self-selection**: a fixed set of sub-panels is built once,
and on each item every sub-panel is *offered* the item and appends itself to the description
list only if it applies. The panel itself knows almost nothing about item kinds.

## State

```text
RECORD ItemInfoPanel EXTENDS Window
  item          : optional<InventoryItem>
  background    : optional<FrameWindow>
  name, weight, cost, trade_note : optional<Label>
  icon          : optional<Picture>
  icon_authored_size : (real, real)
  description   : optional<ScrollView>     # everything below lives inside this
  sub_panels    : condition, weapon parameters, artefact parameters,
                  outfit protections, booster effects, installed upgrades
                  — each optional, each present only if the layout defines it
  show_description_text : bool
  fit_to_height : bool
  complex_desc  : bool           # the reflow variant; see below
  dwell         : int (ms)       # how long a hover must last before this appears
```

**Invariants**

- **Nothing draws while no item is set.** The panel is enabled and disabled with the item.
- Every sub-panel is created once at construction and **re-parented into the description list
  on every item**. The list is cleared first; adding is not ownership-transferring for these.
- A sub-panel whose layout document has no section for it is deleted at construction, and
  every later use is guarded. The layout decides which facts this build of the panel can show.

## Two layout idioms

**Contract** — the panel detects which of two shapes the layout authored and lays out
accordingly. The discriminator is **whether the condition sub-panel is a direct child of the
panel or lives inside the description list**.

| | older idiom | newer idiom |
|---|---|---|
| condition display | a progress bar pinned to the panel | a row inside the description list |
| field positions | authored, left alone | recomputed by stacking measured heights |
| icon grid unit | 50 units, clamped to the authored size | 40 units, unclamped |
| description text | appended **last**, after the sub-panels | appended first |

**Notes** — the same discriminator drives four unrelated decisions, which is why it reads as
a mode and is really an inference from one element's placement. It is how the same panel
serves the first game's data and the later games'.

The description-text ordering difference is the one a player sees: in the older idiom the
numbers come first and the prose last.

A further authored flag, *complex description*, re-enables the measured stacking within the
older idiom for layouts that want it. Three combinations, all shipped.

## Filling in an item

**Contract** —

```text
FUNCTION show(cell_item, compare_item, price, trade_note)
  IF no cell item THEN forget the item; disable; RETURN
  item := the inventory item behind the cell item
  name   := the item's display name, fitted to its text
  weight := the item's weight, with the localized unit
  cost   := the price, hidden when the caller passed the "no price" sentinel
  note   := the localized trade note, hidden when absent
  stack the four vertically, each 4 units under the last   # newer idiom only
  clear the description list
  IF description text is enabled
    append the item's description, word-wrapped and fitted, in complex text mode
  offer the item to each sub-panel in turn; each appends itself if it applies
  IF fitting to height
    shrink the list to its content, then the panel to the list, floor 105 units square
    resize the background frame with it
  scroll the list to the top
  dress the icon from the equipment atlas at the item's grid rectangle
```

**Notes** — the **compare item** threads through to the condition, weapon and outfit
sub-panels, which render each number twice — the item's and the comparison's — so the trade
screen can show what the player would be giving up. The panel itself never compares anything.

**Stacked ammunition reports the stack's weight.** An ammunition box's own weight field is
empty when it is a stack head, so the weight is recomputed as the head's base weight plus
every stacked child's. That is the only place this panel looks at the *cell item* rather than
the inventory item, and it exists because stacking is a UI concept the game's weight model
does not have.

The 105-unit floor on the fitted size stops a one-line tooltip from collapsing to a sliver of
frame.

The icon is scaled horizontally by the runtime aspect factor, and in the older idiom also
clamped to the authored box — chapter 15's non-uniform stretch again.

The price is hidden rather than blanked when absent, and **the whole price and trade-note
block is skipped outside single player**, where a separate path overwrites it.

## The dwell

**Contract** — the panel reads a delay from its layout, defaulting to half a second, and
publishes it. The panel does not implement the delay; whoever shows it does.

**Notes** — the delay belongs to the *layout*, because it is a per-placement decision — a
tooltip over a dense grid wants one, a fixed panel beside a trade window does not. Publishing
it rather than acting on it is what lets the upgrade bench apply the same delay to its own
hover logic. See [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md).

## The sub-panels' self-selection

**Contract** — six offers, each a one-line rule:

| Sub-panel | Applies when |
|---|---|
| condition | the item is a weapon or an outfit |
| weapon parameters | the item's configuration section declares weapon parameters |
| artefact parameters | ... declares artefact parameters |
| outfit protections | the item is an outfit or a helmet |
| installed upgrades | the item has at least one upgrade installed |
| booster effects | the item is edible |

**Notes** — two of the six ask the **configuration section**, not the item's type, and that
is the more flexible rule: a mod can give a non-weapon weapon parameters and they will be
shown. The other four ask the type because there is no configuration key that distinguishes
them.

The upgrade sub-panel is only built at all when an alife simulation exists, because upgrades
are alife-registry state; in its absence the panel simply never shows upgrades.

## Two construction forms

**Contract** — one form reads the layout and takes its own rectangle from the document; the
other is given a position and size by the caller first. The document-driven form returns
**false for an empty document**, which is how a game whose data has no such panel omits it.

**Notes** — the empty-document test is on the root's child count rather than on the load
succeeding, because the loader tolerates an empty file. A rebuild should make absence and
emptiness the same answer.
