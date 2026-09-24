# src/xrGame/ui/ui_af_params.h

> Declares the artefact parameter panel and the signed value row it stacks.

**Needs** — [`ui_af_params.cpp`](ui_af_params.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`../../xrServerEntities/alife_space.h`](../../xrServerEntities/alife_space.h.md)
**Used by** — [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`ui_af_params.cpp`](ui_af_params.cpp.md)
**Tier floor** — T3: declarations of a panel and its row

## Purpose

Declares the surface implemented in [`ui_af_params.cpp`](ui_af_params.cpp.md). The panel holds one
row per possible artefact property — condition, five restoration rates, nine damage immunities, and
carry-capacity — allocated up front and stacked on demand, so that only the properties an artefact
actually has are shown.

The immunity array is declared three shorter than the full damage-type enumeration, which is the
one place the count of *displayable* damage types differs from the count of damage types.

Exported units:

- `CUIArtefactParams` — the panel.
  - `InitFromXml(document)` — build; reports absence so a tooltip can omit the panel.
  - `Check(section)` — whether an item section is an artefact this panel can describe.
  - `SetInfo(item)` — fill and stack from one artefact.
- `UIArtefactParamItem` — one row: a caption, a signed value with a unit, and a colour rule.
