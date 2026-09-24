# src/xrUICore/ui_focus.h

> Declares the directional-navigation registry implemented in [`ui_focus.cpp`](ui_focus.cpp.md), and the nine-direction vocabulary it reasons in.

**Needs** — [`ui_focus.cpp`](ui_focus.cpp.md) · [`ui_defs.h`](ui_defs.h.md) · [`ui_debug.h`](ui_debug.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIMapFilters.cpp`](../xrGame/ui/UIMapFilters.cpp.md) · [`UIButton.cpp`](Buttons/UIButton.cpp.md) · [`UIComboBox.cpp`](ComboBox/UIComboBox.cpp.md) · [`UICustomEdit.cpp`](EditBox/UICustomEdit.cpp.md) · [`UIListBoxItem.cpp`](ListBox/UIListBoxItem.cpp.md) · [`UIListItem.cpp`](ListWnd/UIListItem.cpp.md) · [`UIListWnd.cpp`](ListWnd/UIListWnd.cpp.md) · [`UIPropertiesBox.cpp`](PropertiesBox/UIPropertiesBox.cpp.md) · [`UIFixedScrollBar.cpp`](ScrollBar/UIFixedScrollBar.cpp.md) · [`UIScrollBar.cpp`](ScrollBar/UIScrollBar.cpp.md) · [`UIScrollView.cpp`](ScrollView/UIScrollView.cpp.md) · [`UITrackBar.cpp`](TrackBar/UITrackBar.cpp.md) · [`ui_base.cpp`](ui_base.cpp.md) · [`ui_base.h`](ui_base.h.md) · _and 2 more_
**Tier floor** — T3: a declaration of a registry and an enumeration.

## Purpose

Declares the type whose substance lives in [`ui_focus.cpp`](ui_focus.cpp.md). The decision
stated only here is the ownership one, and the header says it outright: **this registry does
not own the windows it holds**. It stores borrowed references, and a window is responsible
for removing itself before it dies. A rebuild with a safer reference type removes a whole
class of dangling-entry bugs, and the per-frame re-partition means a weak reference that has
expired can simply be dropped.

The other decision here is the read-only intent: the registry holds its windows immutably and
hands out mutable references only at the boundary, to callers that legitimately act on them.
The comment in the source is explicit that the constness is a statement about *this* file's
behaviour, not a guarantee about the caller's.

The type is exported to the script layer, so its method names are part of the frozen script
surface.

## Exported units

- `FocusDirection` — `Same` plus the eight compass directions
- `RegisterFocusable` / `UnregisterFocusable` — join or leave the registry
- `IsRegistered` / `IsValuable` / `IsNonValuable` — membership and eligibility queries
- `Update(root)` — re-partition eligibility against the active screen; once per frame
- `LockToWindow(locker)` / `Unlock()` / `GetLocker()` — restrict eligibility to one subtree
- `GetFocused()` / `SetFocused(window)` — the current focus; setting it warps the cursor
- `FindClosestFocusable(from, direction)` — the primary and fallback navigation candidates
- `DrawDebugInfo(from, to, colour, text_colour)` — the direction overlay
