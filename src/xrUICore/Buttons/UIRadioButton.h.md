# src/xrUICore/Buttons/UIRadioButton.h

> Declares the radio button — a tab button wearing a fixed radio graphic, so that mutual exclusion comes free from the tab group.

**Needs** — [`UIRadioButton.cpp`](UIRadioButton.cpp.md) · [`TabControl/UITabButton.h`](../TabControl/UITabButton.h.md)
**Used by** — [`UIScriptWnd_script.cpp`](../../xrGame/ui/UIScriptWnd_script.cpp.md) · [`UIRadioButton.cpp`](UIRadioButton.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIRadioButton.cpp`](UIRadioButton.cpp.md).

The load-bearing decision is the base class: a radio button *is* a tab button. Mutual
exclusion inside a group is not implemented here at all — the tab control already guarantees
that exactly one of its buttons is pushed, so a group of radio buttons is a tab control whose
buttons happen to look like radio dots.

## Exported units

- `CUIRadioButton` — the radio button.
- `InitButton(pos, size)` — forces the radio texture set and lays the label out beside the dot.
- `InitTexture(name)` — deliberately ignores its argument and succeeds; the texture set is
  fixed.
- `SendMessage` / `OnMouseDown` — translate the tab control's "tab changed" into a radio-set
  notification.
