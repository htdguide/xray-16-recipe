# src/xrGame/ui/UIXmlInit.h

> Declares the game layer's four additions to the chapter-15 layout element vocabulary.

**Needs** — [`UIXmlInit.cpp`](UIXmlInit.cpp.md) · [`xrUICore/XML/UIXmlInitBase.h`](../../xrUICore/XML/UIXmlInitBase.h.md)
**Used by** — [`EliteDetector.cpp`](../EliteDetector.cpp.md) · [`MainMenu.cpp`](../MainMenu.cpp.md) · [`Missile.cpp`](../Missile.cpp.md) · [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`UIGameAHunt.cpp`](../UIGameAHunt.cpp.md) · [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`UIPlayerItem.h`](../UIPlayerItem.h.md) · [`UITeamHeader.h`](../UITeamHeader.h.md) · [`UITeamPanels.h`](../UITeamPanels.h.md) · [`UITeamState.h`](../UITeamState.h.md) · [`encyclopedia_article.cpp`](../encyclopedia_article.cpp.md) · [`map_location.cpp`](../map_location.cpp.md) · [`map_spot.cpp`](../map_spot.cpp.md) · [`ArtefactDetectorUI.cpp`](ArtefactDetectorUI.cpp.md) · _and 75 more_
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UIXmlInit.cpp`](UIXmlInit.cpp.md). Chapter 15's rule is that
the set of layout element types is exactly the set of reader functions, and this type extends that
set by four — which is how the game layer adds widget types to the layout vocabulary without
chapter 15 knowing about them.

Every screen in this directory names *this* type rather than the base, so all readers, inherited and
added, are reachable through one name.

Exported units:

- `CUIXmlInit` — the extended reader set.
- `InitDragDropListEx` — the inventory grid.
- `InitTabButtonMP` — the multiplayer tab button.
- `InitSleepStatic` — the sleep-screen strip.
- `InitHintWindow` — a window carrying a delayed tooltip.
