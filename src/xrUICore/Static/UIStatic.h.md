# src/xrUICore/Static/UIStatic.h

> Declares the picture-and-text widget implemented in [`UIStatic.cpp`](UIStatic.cpp.md), the base of nearly every control in the chapter.

**Needs** — [`UIStatic.cpp`](UIStatic.cpp.md) · [`UIStaticItem.h`](UIStaticItem.h.md) · [`UILanimController.h`](UILanimController.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`EliteDetector.cpp`](../../xrGame/EliteDetector.cpp.md) · [`UIGameCustom_script.cpp`](../../xrGame/UIGameCustom_script.cpp.md) · [`UIZoneMap.h`](../../xrGame/UIZoneMap.h.md) · [`WeaponBinocularsVision.h`](../../xrGame/WeaponBinocularsVision.h.md) · [`encyclopedia_article.h`](../../xrGame/encyclopedia_article.h.md) · [`map_spot.h`](../../xrGame/map_spot.h.md) · [`ArtefactDetectorUI.h`](../../xrGame/ui/ArtefactDetectorUI.h.md) · [`UIBoosterInfo.cpp`](../../xrGame/ui/UIBoosterInfo.cpp.md) · [`UICellItem.h`](../../xrGame/ui/UICellItem.h.md) · [`UIDebugFonts.cpp`](../../xrGame/ui/UIDebugFonts.cpp.md) · [`UIDebugFonts.h`](../../xrGame/ui/UIDebugFonts.h.md) · [`UIDragDropListEx.cpp`](../../xrGame/ui/UIDragDropListEx.cpp.md) · [`UIDragDropReferenceList.cpp`](../../xrGame/ui/UIDragDropReferenceList.cpp.md) · [`UIEditKeyBind.cpp`](../../xrGame/ui/UIEditKeyBind.cpp.md) · _and 69 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIStatic.cpp`](UIStatic.cpp.md). Three decisions are
stated only here.

**A static is three things at once**: a window (it has a place in the tree), a texture owner
(it can be configured from a `<texture>` element), and a colour-animation target. A rebuild
that separates "drawable" from "container" will find every derived control assumes all three.

**The text block is held by reference and created on demand**, and every text operation is a
one-line delegation to it. That delegation layer is why a caller never has to ask whether the
widget has text yet. It is also pure indirection: a rebuild that makes the text block a plain
optional field loses nothing.

**Most of the surface is text and texture delegation, not behaviour.** The methods with real
content are the draw and update pair, the two adjust-to-text operations, and the transform
animation; everything else forwards.

The two animation-state records declared here — a colour container and a transform container
adding an original size — duplicate the pair in
[`UILanimController.h`](UILanimController.h.md). The duplication is an accident of history;
a rebuild keeps one.

## Exported units

Text, all delegating to the block:

- `GetText` / `SetText` / `SetTextST` (the second localizes through the string table)
- `SetFont` / `GetFont` / `SetTextColor` / `GetTextColor`
- `SetTextComplexMode` / `SetTextAlignment` / `SetVTextAlignment`
- `SetEllipsis` / `SetCutWordsMode` / `SetTextOffset`
- `TextItemControl()` — the block itself, created on first call
- `AdjustHeightToText` / `AdjustWidthToText`

Texture, satisfying the texture-owner interface:

- `InitTexture` / `InitTextureEx` / `CreateShader` / `SetShader` / `GetShader`
- `SetTextureColor` / `GetTextureColor` / `SetTextureRect` / `GetTextureRect`
- `SetTextureOffset` / `GetTextureOffeset` / `SetStretchTexture` / `GetStretchTexture`
- `TextureOn` / `TextureOff`
- `GetStaticItem` / `GetUIStaticItem` — the drawable quad, for callers that need it directly
- `SetHeadingPivot` / `ResetHeadingPivot` — rotation anchor

Behaviour:

- `Draw` / `DrawTexture` / `DrawText` — the fixed texture-children-text order
- `Update` — animations and delayed hint
- `OnFocusLost` — release the shared hint
- `SetHeading` / `GetHeading` / `Heading` / `EnableHeading` / `SetConstHeading` /
  `GetConstHeading` — rotation
- `SetXformLightAnim` / `ResetXformAnimation` — the transform animation
- `ColorAnimationSetTextureColor` / `ColorAnimationSetTextColor` — the colour animation sinks
- `m_stat_hint_text` — the delayed hint's content; empty means no hint
