# src/xrUICore/Buttons/UI3tButton.h

> Declares the button the shipped games actually use — one whose background and text colour are chosen from four visual states rather than drawn by an offset.

**Needs** — [`UI3tButton.cpp`](UI3tButton.cpp.md) · [`UIButton.h`](UIButton.h.md) · [`InteractiveBackground/UI_IB_Static.h`](../InteractiveBackground/UI_IB_Static.h.md) · [`InteractiveBackground/UIInteractiveBackground.h`](../InteractiveBackground/UIInteractiveBackground.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`ChangeWeatherDialog.cpp`](../../xrGame/ui/ChangeWeatherDialog.cpp.md) · [`ServerList.h`](../../xrGame/ui/ServerList.h.md) · [`UIDemoPlayControl.cpp`](../../xrGame/ui/UIDemoPlayControl.cpp.md) · [`UIInventoryUpgradeWnd.cpp`](../../xrGame/ui/UIInventoryUpgradeWnd.cpp.md) · [`UIKickPlayer.cpp`](../../xrGame/ui/UIKickPlayer.cpp.md) · [`UIMPAdminMenu.cpp`](../../xrGame/ui/UIMPAdminMenu.cpp.md) · [`UIMPChangeMapAdm.cpp`](../../xrGame/ui/UIMPChangeMapAdm.cpp.md) · [`UIMPPlayersAdm.cpp`](../../xrGame/ui/UIMPPlayersAdm.cpp.md) · [`UIMPServerAdm.cpp`](../../xrGame/ui/UIMPServerAdm.cpp.md) · [`UIMPServerAdm.h`](../../xrGame/ui/UIMPServerAdm.h.md) · [`UIMapLegend.cpp`](../../xrGame/ui/UIMapLegend.cpp.md) · [`UIMapList.cpp`](../../xrGame/ui/UIMapList.cpp.md) · [`UIMapWnd2.cpp`](../../xrGame/ui/UIMapWnd2.cpp.md) · [`UISecondTaskWnd.cpp`](../../xrGame/ui/UISecondTaskWnd.cpp.md) · _and 18 more_
**Tier floor** — T3: state selection and two sound handles.

## Purpose

Declares the surface implemented in [`UI3tButton.cpp`](UI3tButton.cpp.md). The name is
historical — "three-texture" — but there are four states: enabled, disabled, highlighted and
touched. Almost every button in the shipped XML is this class; the plain
[`CUIButton`](UIButton.h.md) is rarely instantiated directly.

## Exported units

- `CUI3tButton` — the four-state button.
- `InitButton(pos, size)` — creates the background child, either a plain quad background or a
  three-segment stretched line background, depending on the frame-line mode flag set before
  the call.
- `InitTexture(base_name)` — derives four texture names from one base by appending `_e`,
  `_d`, `_t`, `_h`. This suffix convention is **frozen**: the shipped texture description
  files name their entries this way.
- `InitTexture(enabled, disabled, touched, highlighted)` — the explicit four-name form.
- `InitSoundH` / `InitSoundT` — the hover sound and the click sound.
- `SetStateTextColor(color, state)` — per-state text colour, with a per-state "use it" flag.
- `m_frameline_mode` / `vertical` — chooses and orients the stretched-line background.
- `m_background` / `m_back_frameline` — exactly one of these is non-null after init.

**Notes** — the two background kinds are not polymorphic through a common pointer; every
method that touches the background tests which one exists. A rebuild should make this one
interface with two implementations — the ten-odd `if background else if frameline` pairs in
the implementation are all the same branch.
