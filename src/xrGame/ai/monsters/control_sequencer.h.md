# src/xrGame/ai/monsters/control_sequencer.h

> Declares the clip-list player and its payload, implemented in [`control_sequencer.cpp`](control_sequencer.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_sequencer.cpp`](control_sequencer.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CAnimationSequencer`. Substance is in
[`control_sequencer.cpp`](control_sequencer.cpp.md).

## State

`SAnimationSequencerData` — the channel payload: an ordered list of clips. Emptied on every
capture.

Exported units: `reset_data`, `on_capture`, `activate`, `on_release`, `on_event`,
`check_start_conditions`. The seizure happens in `on_capture` rather than `activate`, which
is what lets a caller fill the list while the creature already stands still.
