# src/xrEngine/line_edit_control.h

> Declares the single-line text editor, its modifier vocabulary and its four modes.

**Needs** — [`line_edit_control.cpp`](line_edit_control.cpp.md) · [`edit_actions.h`](edit_actions.h.md) · [`xr_input.h`](xr_input.h.md)
**Used by** — [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`edit_actions.cpp`](edit_actions.cpp.md) · [`edit_actions.h`](edit_actions.h.md) · [`line_edit_control.cpp`](line_edit_control.cpp.md) · [`UICustomEdit.cpp`](../xrUICore/EditBox/UICustomEdit.cpp.md) · [`UICustomEdit.h`](../xrUICore/EditBox/UICustomEdit.h.md)
**Tier floor** — T2: a declaration plus two small enumerations.

## Purpose

Declares the surface implemented in
[`line_edit_control.cpp`](line_edit_control.cpp.md), and the two enumerations its callers
name.

## Exported units

- **Modifier set** — the six modifier keys as individual left/right flags, plus the three
  combined forms (either shift, either control, either alt), plus caps lock, plus two
  sentinels: *none*, which a handler uses to say "match unconditionally", and *any*. Left
  and right are tracked separately because a handler may require one specifically; the
  combined forms are what almost every handler actually asks for.
- **Mode** — standard, number-only, read-only, file-name. Chosen at initialization; decides
  both which keys are bound and which characters are admitted.
- **`line_edit_control`** — construct with a buffer size; re-initialize with a size and a
  mode. Feed it key press, hold, release and text-input events, and a per-frame tick.
  Register and remove per-key handlers. Read back the whole line, the four display slices,
  the caret blink phase, and whether anything changed recently. Set the line directly, and
  turn selection off for a control that should not have one.
- **`remove_spaces(text)`** — collapse and trim runs of spaces in place.
- **`split_cmd(first, rest, line)`** — split a line at its first space.
- **`g_console_sensitive`** — the key auto-repeat interval in seconds, default 0.15. The
  initial delay before repeat begins is five times it.
