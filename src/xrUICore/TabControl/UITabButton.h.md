# src/xrUICore/TabControl/UITabButton.h

> Declares the tab button implemented in [`UITabButton.cpp`](UITabButton.cpp.md).

**Needs** — [`UITabButton.cpp`](UITabButton.cpp.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md)
**Used by** — [`UIBuyWeaponTab.cpp`](../../xrGame/ui/UIBuyWeaponTab.cpp.md) · [`UITabButtonMP.cpp`](../../xrGame/ui/UITabButtonMP.cpp.md) · [`UITabButtonMP.h`](../../xrGame/ui/UITabButtonMP.h.md) · [`UIRadioButton.cpp`](../Buttons/UIRadioButton.cpp.md) · [`UIRadioButton.h`](../Buttons/UIRadioButton.h.md) · [`UITabButton.cpp`](UITabButton.cpp.md) · [`UITabControl.cpp`](UITabControl.cpp.md) · [`UITabControl.h`](UITabControl.h.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UITabButton.cpp`](UITabButton.cpp.md): a three-state
button carrying a name, which it matches against the group's tab-changed announcement to
decide whether it should be depressed.

The name is public and mutable, and it is the *only* identity a tab has — the control looks
tabs up by it, the options protocol saves it as the remembered tab, and equality between a tab
and a name is defined for it so the group can be searched directly. A rebuild should keep the
name as the identity rather than introducing an index: the shipped settings store tab names.

## Exported units

- `CUITabButton` — the tab.
- `m_btn_id` — the tab's name.
- `IsIdDefaultAssigned` — whether the name was invented at load time because the layout gave
  none, which lets the XML reader distinguish an authored name from a generated one.
- `SendMessage` — the group's exclusivity rule, applied to this tab.
- `OnMouseAction` — deliberately bypasses the inherited press machine.
- `OnMouseDown` — raises the tab-changed announcement on press.
