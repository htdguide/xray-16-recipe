# src/xrGame/ui/UIMMShniaga.h

> Declares the main-menu drum, its three pages, and the lens that draws outside the widget tree.

**Needs** — [`UIMMShniaga.cpp`](UIMMShniaga.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`MMSound.h`](MMSound.h.md)
**Used by** — [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`UIMMShniaga.cpp`](UIMMShniaga.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMMShniaga.cpp`](UIMMShniaga.cpp.md).

## `CUIMMMagnifer`

**Contract** — the lens over the drum. Its whole contract is the mode switch: in
post-process mode it registers with the main menu's after-scene draw pass and removes itself
from ordinary drawing; leaving that mode unregisters it. Registration is released on
destruction, and failing to do so leaves the pass holding a dead widget.

**Notes** — this is the one widget in the chapter that is drawn by something other than its
parent. See the implementation twin for why, and for the consequence that hiding it means
moving it off the canvas.

## `CUIMMShniaga`

**Contract** — the drum. It is also a **device-reset listener**, though it does nothing on
reset; the registration is vestigial and a rebuild drops it.

```text
ENUM Page
  main
  new_game
  new_network_game
  none              # before construction completes
```

- `InitShniaga` — read the band, the lens, the two cogs, the two gratings, the caption view
  and the authored rest offset, then build the three caption lists from game state and show
  the main one, then start the menu music.
- `SetPage(page, document, path)` — replace a page's caption list from another document; how
  the game swaps in a different sub-menu.
- `ShowPage` / `ShowMain` / `ShowNewGame` / `ShowNetworkGame` — switch pages.
- `SetVisibleMagnifier` — park the lens off canvas or bring it back.
- `Update` — drive the motion, the cogs and the sound.
- the three input handlers, `SendMessage` and `OnBtnClick`.

**Notes** — the caption lists are three separate members rather than an array indexed by page,
so every operation over them is written three times and the page enumeration is compared
against bare integers in two places. A rebuild indexes an array by the page and the triplication
disappears. One helper begins with an assertion of a constant that is always true — a
disabled check, not a live one.
