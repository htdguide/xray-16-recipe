# src/xrUICore/EditBox/UIEditBox.h

> Declares the entry field dressed in a stretched three-segment frame and bound to a string console variable.

**Needs** — [`UIEditBox.cpp`](UIEditBox.cpp.md) · [`UICustomEdit.h`](UICustomEdit.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md)
**Used by** — [`ServerList.h`](../../xrGame/ui/ServerList.h.md) · [`UICDkey.h`](../../xrGame/ui/UICDkey.h.md) · [`UIChatWnd.cpp`](../../xrGame/ui/UIChatWnd.cpp.md) · [`UIMPServerAdm.cpp`](../../xrGame/ui/UIMPServerAdm.cpp.md) · [`UITalkWnd.h`](../../xrGame/ui/UITalkWnd.h.md) · [`UIComboBox.h`](../ComboBox/UIComboBox.h.md) · [`UIEditBox.cpp`](UIEditBox.cpp.md) · [`UIMessageBox.cpp`](../MessageBox/UIMessageBox.cpp.md) · [`UIMessageBox.h`](../MessageBox/UIMessageBox.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIEditBox.cpp`](UIEditBox.cpp.md). This is the entry
field the XML vocabulary instantiates: a [`CUICustomEdit`](UICustomEdit.h.md) that also has a
visible border and can act as a settings-screen control.

## Exported units

- `CUIEditBox` — the framed entry field.
- `InitCustomEdit(pos, size)` — geometry, propagated to the border.
- `InitTexture` / `InitTextureEx` — create the border on first use and load its texture. The
  border is created lazily, so a field whose XML names no texture has no border child at all.
- The five settings-item operations, over a string-valued console variable.

**Notes** — the two base classes are inherited in the order settings-item first, widget
second. That ordering matters only to the source language; what survives is that the same
object answers to both protocols.
