# src/xrUICore/Windows/UIFrameLineWnd.h

> Declares the three-piece stretchable line implemented in [`UIFrameLineWnd.cpp`](UIFrameLineWnd.cpp.md).

**Needs** — [`UIFrameLineWnd.cpp`](UIFrameLineWnd.cpp.md) · [`UIWindow.h`](UIWindow.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ArtefactDetectorUI.h`](../../xrGame/ui/ArtefactDetectorUI.h.md) · [`UIActorInfo.cpp`](../../xrGame/ui/UIActorInfo.cpp.md) · [`UIFactionWarWnd.cpp`](../../xrGame/ui/UIFactionWarWnd.cpp.md) · [`UIKeyBinding.h`](../../xrGame/ui/UIKeyBinding.h.md) · [`UIMapList.cpp`](../../xrGame/ui/UIMapList.cpp.md) · [`UITalkDialogWnd.h`](../../xrGame/ui/UITalkDialogWnd.h.md) · [`UIBtnHint.cpp`](../Buttons/UIBtnHint.cpp.md) · [`UIEditBox.cpp`](../EditBox/UIEditBox.cpp.md) · [`UIEditBox.h`](../EditBox/UIEditBox.h.md) · [`UIInteractiveBackground.h`](../InteractiveBackground/UIInteractiveBackground.h.md) · [`UIListBoxItem.h`](../ListBox/UIListBoxItem.h.md) · [`UIListWnd.cpp`](../ListWnd/UIListWnd.cpp.md) · [`UIListWnd.h`](../ListWnd/UIListWnd.h.md) · [`UIFixedScrollBar.cpp`](../ScrollBar/UIFixedScrollBar.cpp.md) · _and 8 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIFrameLineWnd.cpp`](UIFrameLineWnd.cpp.md). The
declaration fixes the vocabulary the rest of the toolkit uses for segmented art: a line is
exactly three pieces, named *first*, *back* and *second*, where first and second are the caps
— left and right when horizontal, top and bottom when vertical — and back is the piece that
tiles. Several widgets reach in and set individual segments, so the three-way split is public,
not an implementation detail.

The declaration also records where this type does not fit the toolkit's texture-owner
interface: that interface assumes one texture rectangle per widget, and here the single-rect
setter is deliberately inert while the getter answers the first cap. A rebuild should give
segmented art its own interface rather than half-implementing the flat one.

## Exported units

- `CUIFrameLineWnd` — the line.
- `RectSegment` — the three-piece vocabulary: first cap, tiling middle, second cap.
- `InitTexture(name)` / `InitTextureEx(name, shader)` — load all three by suffixing the base
  name; returns false rather than failing when the caller asked for non-fatal, which is how
  callers probe for an older asset generation.
- `SetTextureRect(rect, segment)` — write one segment's source rectangle.
- `SetShader` — set all three segments' shaders at once.
- `SetTextureVisible` — force the line drawn or not, independently of whether it loaded.
- `SetTextureColor` / `GetTextureColor` — the tint every piece is drawn with.
- `SetStretchTexture` / `GetStretchTexture` — fill the widget's short dimension, or draw at
  the art's natural thickness.
- `IsHorizontal` / `SetHorizontal` — the orientation, which selects which dimension tiles.
- `GetTitleText(create_on_demand)` — an optional caption, created only when asked for.
- `Draw`.
