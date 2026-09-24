# src/xrGame/ui/UIMpTradeWnd_init.cpp

> The buy screen's construction: the category tree, the tab strip built from the tree's own
> buttons, the ten lists, thirty-odd controls, and the price table.

**Needs** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UITabButtonMP.h`](UITabButtonMP.h.md) · [`UIBuyWeaponTab.h`](UIBuyWeaponTab.h.md) · [`xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md)
**Used by** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T3.

## Purpose

One long construction, separated because it is long. The decisions in it are about *where things
come from*, not about behaviour.

## `Init`

**Contract** — build the category tree's shape from the buy layout document and its contents
from the named item section; apply the document to the screen; build the tab strip by adopting
the tree's own depth-one buttons; build the shop panel and the ten lists, binding drag
behaviour to each; build the buttons, the money and rank readouts, the two message lines, the
item detail panel and the three item tint colours; load the price table from the named price
section; and finish at rank zero with the store at its root.

```text
FUNCTION init(item_section, price_section)
  doc <- load_layout("mp_buy_menu.xml")
  store <- tree from doc at "items_hierarchy"
  store.fill_items_from(item_section)
  apply(doc, "main", self)

  tab_strip <- from doc at "tab_control"
  FOR EACH child OF store.root: tab_strip.adopt(child.button)   # NOT owned by the strip
  tab_strip.clear_selection()

  shop_panel <- from doc at "shop_wnd"
  FOR i, element IN the ten list element names
    list[i] <- drag_drop_list from doc at element
    IF i IS NOT shop THEN adopt it into the screen     # the shop list is owned by the panel
    bind drag handlers to list[i]

  ... the buttons, their bindings, the readouts, the colours ...
  item_info <- detail panel from "buy_menu_item.xml", placed at the origin at 100 by 100
  price_table <- load(price_section)
  set_rank(0);  update_shop();  set_current_item(none)
```

**Notes**

- **The tab strip does not own its buttons and the shop list is not a child of the screen.**
  Both belong to something with a longer life — the category tree, and the shop panel that is
  emptied every navigation — so ownership is explicitly withheld at construction and released
  by hand at teardown. A rebuild with shared ownership deletes this whole concern.
- The sub-category buttons are bound **by name**, not by identity: one binding on the name
  `sub_btn` catches every one of them, for two different notifications, because the buttons are
  created by the tree and the screen never sees them individually. This is the toolkit's
  bind-by-name facility earning its place.
- **Seven attachment buttons are declared and three are commented out.** The pistol-ammunition,
  rifle-ammunition and second-rifle-ammunition buttons exist in the enumeration, have handlers,
  and are reachable *only by keyboard* — see [`_misc`](UIMpTradeWnd_misc.cpp.md). They were
  removed from the layout and their keyboard path was left. A rebuild either restores the
  buttons or drops the keys.
- The detail panel's placement is nominal — origin, a hundred square — because its own document
  overrides both.
