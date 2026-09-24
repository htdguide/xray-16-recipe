# src/xrUICore/Static/UIStaticItem.h

> Declares the drawable quad implemented in [`UIStaticItem.cpp`](UIStaticItem.cpp.md), and the mirroring vocabulary the layout format uses.

**Needs** — [`UIStaticItem.cpp`](UIStaticItem.cpp.md) · [`ui_defs.h`](../ui_defs.h.md)
**Used by** — [`HitMarker.cpp`](../../xrGame/HitMarker.cpp.md) · [`UIFrameRect.cpp`](../../xrGame/UIFrameRect.cpp.md) · [`UIFrameRect.h`](../../xrGame/UIFrameRect.h.md) · [`UIArtefactPanel.h`](../../xrGame/ui/UIArtefactPanel.h.md) · [`UIFrameLine.cpp`](../../xrGame/ui/UIFrameLine.cpp.md) · [`UIFrameLine.h`](../../xrGame/ui/UIFrameLine.h.md) · [`UIStatic.cpp`](UIStatic.cpp.md) · [`UIStatic.h`](UIStatic.h.md) · [`UIStaticItem.cpp`](UIStaticItem.cpp.md) · [`UITextureMaster.cpp`](../XML/UITextureMaster.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UIStaticItem.cpp`](UIStaticItem.cpp.md). One decision is
made only here, and it is the important one: **the item is a value with public fields, not an
encapsulated object**. Owning widgets set its position and size directly every frame. That is
deliberate — it is the leaf of the draw path and every accessor would be on the per-frame
hot path — and a rebuild is free to encapsulate it, provided the per-frame update stays
allocation-free and branch-light.

The second decision here is the **validity flags**. Size and texture rectangle are each
either authored or "derive it from the page when you first need it", and the flags are what
distinguish the two. A rebuild expresses them as optional fields and the flags vanish.

## Exported units

- `EUIMirroring` — `None`, `Horisontal`, `Vertical`, `Both`; the values a layout's `mirror`
  attribute maps onto
- `CreateShader(texture, pass)` / `SetShader(material)` / `GetShader()` — the material; the
  first invalidates the derived size and rectangle, the second does not
- `Init(texture, pass, x, y)` — create a material and place the item
- `Render()` / `Render(angle)` — emit the quad, unrotated or rotated
- `SetPos` / `GetPosX` / `GetPosY` / `SetSize` / `GetSize`
- `SetTextureRect` / `GetTextureRect` — in texels of the page
- `SetTextureColor` / `GetTextureColor` — the modulation colour
- `SetMirrorMode` / `GetMirrorMode`
- `SetHeadingPivot(pivot, offset, pin_top_left)` / `ResetHeadingPivot()` /
  `GetHeadingPivot()` / `GetFixedLTWhileHeading()` — the rotation anchor, and whether the
  item's top-left stays put while it turns
