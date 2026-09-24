# src/xrUICore/Windows/UIFrameWindow.h

> Declares the nine-piece framed panel implemented in [`UIFrameWindow.cpp`](UIFrameWindow.cpp.md).

**Needs** — [`UIFrameWindow.cpp`](UIFrameWindow.cpp.md) · [`UIWindow.h`](UIWindow.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ServerList.h`](../../xrGame/ui/ServerList.h.md) · [`UIInvUpgradeInfo.cpp`](../../xrGame/ui/UIInvUpgradeInfo.cpp.md) · [`UIItemInfo.cpp`](../../xrGame/ui/UIItemInfo.cpp.md) · [`UIKeyBinding.h`](../../xrGame/ui/UIKeyBinding.h.md) · [`UIKickPlayer.cpp`](../../xrGame/ui/UIKickPlayer.cpp.md) · [`UIMapLegend.cpp`](../../xrGame/ui/UIMapLegend.cpp.md) · [`UIMapList.cpp`](../../xrGame/ui/UIMapList.cpp.md) · [`UIMapWnd.cpp`](../../xrGame/ui/UIMapWnd.cpp.md) · [`UISecondTaskWnd.cpp`](../../xrGame/ui/UISecondTaskWnd.cpp.md) · [`UITalkWnd.h`](../../xrGame/ui/UITalkWnd.h.md) · [`UIVoteStatusWnd.h`](../../xrGame/ui/UIVoteStatusWnd.h.md) · [`UIXmlInit.cpp`](../../xrGame/ui/UIXmlInit.cpp.md) · [`map_hint.h`](../../xrGame/ui/map_hint.h.md) · [`UIBtnHint.cpp`](../Buttons/UIBtnHint.cpp.md) · _and 12 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIFrameWindow.cpp`](UIFrameWindow.cpp.md), and fixes the
nine-piece vocabulary: an interior, four edges and four corners, all loaded from one base name
by suffix and all sharing one shader. The single shader is the decision that distinguishes it
from [the frame line](UIFrameLineWnd.h.md), which allows a shader per segment — a frame's
nine pieces must live in one atlas, and in exchange the whole panel is always one draw.

The declaration also states the one constraint the type imposes on its caller: it overrides
the size setter, and a framed panel cannot be made smaller than its own corners.

## Exported units

- `CUIFrameWindow` — the framed panel.
- `EFramePart` — the nine-piece vocabulary.
- `InitTexture(name)` / `InitTextureEx(name, shader)` — load all nine by suffix, checking that
  the corners and edges agree on the dimensions they share; returns whether every piece was
  found.
- `SetWndSize` — clamps up to the frame's minimum when a texture is loaded.
- `SetTextureColor` / `GetTextureColor` — the tint.
- `GetTitleText(create_on_demand)` — an optional caption.
- `Draw`.
- `SetTextureRect` / `SetStretchTexture` / `GetStretchTexture` — present to satisfy the
  texture-owner interface and deliberately inert; a nine-piece frame has no single rectangle
  and never stretches a piece. `GetTextureRect` answers the interior's rectangle.
