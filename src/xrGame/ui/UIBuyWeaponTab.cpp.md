# src/xrGame/ui/UIBuyWeaponTab.cpp

> The buy menu's tab strip: the same tabs as everywhere else, minus the toolkit's guard
> against re-selecting the tab you are already on.

**Needs** — [`UIBuyWeaponTab.h`](UIBuyWeaponTab.h.md) · [`../../xrUICore/TabControl/UITabButton.h`](../../xrUICore/TabControl/UITabButton.h.md)
**Used by** — [`UIBuyWeaponTab.h`](UIBuyWeaponTab.h.md)
**Tier floor** — T3: one notification override

## Purpose

The whole file exists for one behavioural difference, and stating it is the whole content of
the page.

## `SendMessage`

**Contract** — On a tab-changed notification, find the tab that sent it, record it as pushed,
and fire the change callback — **unconditionally**, including when the tab was already the
selected one. Every other notification is passed to the base.

```text
FUNCTION on_message(sender, message, payload)
  IF message is not "tab changed" THEN delegate to base; RETURN

  FOR EACH tab IN tabs
    IF tab = sender
      pushed = tab.id
      on_tab_change(pushed, previously_pushed)     # fired even if unchanged
      previously_pushed = pushed
      BREAK
```

**Invariants** — The toolkit's tab control suppresses the callback when the new tab equals the
old one. The buy menu needs it fired anyway, because selecting the current tab is how the
player **returns to that tab's item list after drilling into a weapon's attachments** — the
tab has not changed but the page below it has.

That is the only reason this class exists, and a rebuild whose tab control does not suppress
the redundant callback does not need it at all.
