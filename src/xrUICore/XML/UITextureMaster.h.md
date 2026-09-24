# src/xrUICore/XML/UITextureMaster.h

> Declares the icon registry implemented in [`UITextureMaster.cpp`](UITextureMaster.cpp.md), and the two records it is built from.

**Needs** — [`UITextureMaster.cpp`](UITextureMaster.cpp.md) · [`ui_defs.h`](../ui_defs.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ScriptXMLInit.cpp`](../../xrGame/ScriptXMLInit.cpp.md) · [`UIFrameRect.cpp`](../../xrGame/UIFrameRect.cpp.md) · [`map_spot.cpp`](../../xrGame/map_spot.cpp.md) · [`UIEditKeyBind.cpp`](../../xrGame/ui/UIEditKeyBind.cpp.md) · [`UIFrameLine.cpp`](../../xrGame/ui/UIFrameLine.cpp.md) · [`UIHudStatesWnd.cpp`](../../xrGame/ui/UIHudStatesWnd.cpp.md) · [`UIListItemServer.cpp`](../../xrGame/ui/UIListItemServer.cpp.md) · [`UILoadingScreen.cpp`](../../xrGame/ui/UILoadingScreen.cpp.md) · [`UILoadingScreenHardcoded.h`](../../xrGame/ui/UILoadingScreenHardcoded.h.md) · [`UISleepStatic.cpp`](../../xrGame/ui/UISleepStatic.cpp.md) · [`UIStatsIcon.cpp`](../../xrGame/ui/UIStatsIcon.cpp.md) · [`UIXmlInit.cpp`](../../xrGame/ui/UIXmlInit.cpp.md) · [`UIComboBox.cpp`](../ComboBox/UIComboBox.cpp.md) · [`UIScrollBar.cpp`](../ScrollBar/UIScrollBar.cpp.md) · _and 9 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`UITextureMaster.cpp`](UITextureMaster.cpp.md). Two
decisions live only here.

**The registry is process-wide state, not an object anyone holds.** Every widget reaches it
by name, from any thread of the load path, without a reference. That is why a UI reset has to
clear it explicitly rather than simply dropping an owner, and why a rebuild that makes it an
injected service has to thread it through every widget's initialisation.

**The material cache is keyed by a pair**, (page, pass), and the pair's ordering is what a
map needs to store it. The ordering as written compares the page names and, only when the
first name does not sort before the second, falls back to comparing pass names — which is not
a strict weak ordering, so two distinct keys can compare equal in both directions. In
practice the UI uses one or two passes and the collision does not bite. A rebuild uses a
proper lexicographic pair comparison, or a hash of the two names.

The registry is exported to scripts, so its lookup names are part of the frozen script
surface.

## Exported units

Records:

- `TEX_INFO` — an icon's page name and its sub-rectangle in that page's texels
- `sh_pair` — the material cache key: page name plus pass name

Operations, all on the process-wide registry:

- `ParseShTexInfo(file)` / `ParseShTexInfo(path, file)` / `ParseShTexInfo(document, override)`
  — load icon descriptions in the single-page or multi-page document shape
- `FreeTexInfo()` / `FreeCachedShaders()` — drop the registry, or just the material cache
- `InitTexture(icon, item, pass)` — resolve into a drawable item, sizing it to the icon
- `InitTexture(icon, pass, out_material, out_rect)` — resolve into loose outputs
- `FindItem` — three forms: asserting, reporting, and reporting with a fallback name
- `ItemExist(icon)` — membership, used to probe for one of several art sets
- `GetTextureRect` / `GetTextureFileName` — asserting accessors
- `GetTextureWidth` / `GetTextureHeight` — each in an asserting and a reporting form
- `GetTextureShader(icon, out)` — a material for the icon's page, bypassing the cache
