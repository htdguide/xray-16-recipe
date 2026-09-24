# src/xrGame/ui/UIHelper.h

> Declares the sixteen construct-configure-adopt creators every screen in this chapter is assembled from.

**Needs** — [`UIHelper.cpp`](UIHelper.cpp.md) · [`UIXmlInit.h`](UIXmlInit.h.md)
**Used by** — [`UIGameAHunt.cpp`](../UIGameAHunt.cpp.md) · [`UIGameCTA.cpp`](../UIGameCTA.cpp.md) · [`UIGameDM.cpp`](../UIGameDM.cpp.md) · [`UIZoneMap.cpp`](../UIZoneMap.cpp.md) · [`map_spot.cpp`](../map_spot.cpp.md) · [`UIAchievements.cpp`](UIAchievements.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md) · [`UIBoosterInfo.cpp`](UIBoosterInfo.cpp.md) · [`UICellItem.cpp`](UICellItem.cpp.md) · [`UIChangeMap.cpp`](UIChangeMap.cpp.md) · [`UICharacterInfo.cpp`](UICharacterInfo.cpp.md) · [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · _and 36 more_
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIHelper.cpp`](UIHelper.cpp.md). It is a namespace of
free functions wearing a class, and a rebuild should make it a namespace: nothing here has
state, and the type is never instantiated.

## Exported units

`CreateNormalWindow`, `CreateStatic`, `CreateScrollView`, `CreateEditBox`,
`CreateProgressBar`, `CreateProgressShape`, `CreateFrameLine`, `CreateFrameWindow`,
`Create3tButton`, `CreateCheck`, `CreateListBox`, `CreateHint`, `CreateDragDropListEx`,
`CreateDragDropReferenceList` — each taking a layout document, an element path, an optional
element index, an optional parent, and a flag saying whether the element's absence is an
error. The set of creators is a subset of chapter 15's reader list: it covers the control
types the *game's* screens use, not every control the toolkit has.
