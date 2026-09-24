# src/xrGame/DemoPlay_Control.h

> Declares the event-seeking demo playback controller implemented in [`DemoPLay_Control.cpp`](DemoPLay_Control.cpp.md).

**Needs** — [`Message_Filter.h`](Message_Filter.h.md)
**Used by** — [`DemoPLay_Control.cpp`](DemoPLay_Control.cpp.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`UIDemoPlayControl.cpp`](ui/UIDemoPlayControl.cpp.md) · [`UIDemoPlayControl.h`](ui/UIDemoPlayControl.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the six watchable actions, the three-state mode, and the two arming entry points.
Substance in [`DemoPLay_Control.cpp`](DemoPLay_Control.cpp.md).

Exported units:

- `demoplay_control` — one armed watch at a time, over a demo's recorded message stream.
- `EAction` — round start (unqualified), kill (by killer's name), die (by victim's name),
  artefact delivered / captured / lost (each by the acting player's name).
- `pause_on`, `cancel_pause_on` — watch at the current playback rate; disarm.
- `rewind_until`, `stop_rewind` — watch at eight times speed with an optional completion
  callback; disarm and restore the rate.
- `no_user_callback` — the shared empty callback used as the default.
- `activate_filer`, `deactivate_filter`, `process_action` — private: install and remove
  the subscription; the shared match path that restores the rate, pauses, disarms and
  calls back.
- One private handler per action, each encoding that event's field layout well enough to
  reach the player identifier it cares about.

**Notes** — the file name differs in capitalization between the header and its
implementation (`DemoPlay` against `DemoPLay`). On a case-sensitive filesystem the include
resolves only because the header's own name is the one spelled in the include directive.
