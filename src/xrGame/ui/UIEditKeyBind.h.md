# src/xrGame/ui/UIEditKeyBind.h

> Declares the key-binding cell: a label that is also a settings control over one binding slot.

**Needs** — [`UIEditKeyBind.cpp`](UIEditKeyBind.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Options/UIOptionsItem.h`](../../xrUICore/Options/UIOptionsItem.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md)
**Used by** — [`UIEditKeyBind.cpp`](UIEditKeyBind.cpp.md) · [`UIKeyBinding.cpp`](UIKeyBinding.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIEditKeyBind.cpp`](UIEditKeyBind.cpp.md). The cell is
constructed with two facts it never changes: whether it edits the *primary* or *secondary*
slot, and whether it edits keyboard or gamepad bindings. Those two bits select one of three
binding slots per action.

## Exported units

- **`CUIEditKeyBind`**
  - `InitKeyBind` — place, dress and disarm the cell.
  - `AssignProps` — resolve the settings entry name to a game action; this is what binds the
    cell to a row of the controls table.
  - the settings protocol: `SetCurrentOptValue`, `SaveBackUpOptValue`, `SaveOptValue`,
    `UndoOptValue`, `IsChangedOptValue`.
  - `SetValue` — re-read and re-display; called both by the protocol and by the page when
    the binding table changes behind its back.
  - `OnMessage` — receive a peer's "action=key" announcement and stand down if in conflict.
  - `SetEditMode` — arm or disarm, taking or releasing keyboard capture.
  - `SetText` — display, shortened to fit, or three dashes when empty.
  - the three input handlers, one per channel: pointer, key, controller axis.

## `CutStringByLength`

**Contract** — a free function, exported by definition rather than by declaration: shorten a
string until it measures no wider than a given width in screen units, using the font's own
cut position for multi-byte fonts and trailing-character removal otherwise. Returns the
resulting length.

**Notes** — it lives here because this is the first place that needed it, not because it
belongs here. A rebuild puts it with the text engine.
