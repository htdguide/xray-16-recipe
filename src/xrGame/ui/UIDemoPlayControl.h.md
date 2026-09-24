# src/xrGame/ui/UIDemoPlayControl.h

> Declares the demo transport bar and the six rewind predicates it offers.

**Needs** — [`UIDemoPlayControl.cpp`](UIDemoPlayControl.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md) · [`DemoPlay_Control.h`](../DemoPlay_Control.h.md)
**Used by** — [`UIGameMP.cpp`](../UIGameMP.cpp.md) · [`game_cl_capture_the_artefact.cpp`](../game_cl_capture_the_artefact.cpp.md) · [`UIDemoPlayControl.cpp`](UIDemoPlayControl.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIDemoPlayControl.cpp`](UIDemoPlayControl.cpp.md).

## State shared with the layout

```text
ENUM RewindKind          # the values are menu row tags, see the .cpp
  until_round_start      # takes no subject
  until_kill
  until_death
  until_artefact_taken
  until_artefact_dropped
  until_artefact_delivered
```

## Exported units

- **`CUIDemoPlayControl`** — the transport screen.
  - `Init` — load the layout, bind the handlers, resolve the playback engine.
  - `Update` — recompose the status line and drive the progress bar.
  - `SendMessage` — route widget notifications through the callback table first, then
    upward.
  - `OnKeyboardAction` — the close binding.
  - `WorkInPause` — always true; the screen is most useful while paused.
  - `GetLastCursorPos` — where the pointer was when the screen last closed, so the holder
    can restore it.
  - the six handlers: restart, speed down, play/pause, speed up, open rewind menu, repeat
    last rewind; plus the two menu-selection handlers.

**Notes** — the player-name list is held in a caller-provided block of storage rather than a
growing container, because its length is known from the demo header before the first name is
read. That is an allocation choice, not a decision: a rebuild reserves capacity and moves on.
