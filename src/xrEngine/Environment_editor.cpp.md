# src/xrEngine/Environment_editor.cpp

> The in-game weather editor: live editing of the active weather frames, with immediate visual feedback and a save back to shipped-format configuration.

**Needs** — [`Environment.h`](Environment.h.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`editor_helper.h`](editor_helper.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: form layout over the records defined elsewhere. Compiled out of a shipping build entirely.

## Purpose

Weather is authored by looking at it. Every field of a weather frame is a number whose meaning is only apparent in the rendered scene, so the editor is a panel inside the running game rather than a separate tool: change a fog colour and the world in front of you changes that frame.

This is the entire reason the environment is declared as an editor tool in [`Environment.h`](Environment.h.md), and it shapes one thing outside itself — a frame must be editable in place, which is why frames are mutable records rather than immutable loaded data.

## `show_frame_parameters`

**Contract** — Draws one weather frame's fields as an editable form, grouped into collapsible categories, with per-field help text. Nothing here decides anything the frame record does not already define; the field list is the record's field list. Editing writes directly into the live frame, so a change is visible on the next interpolation without a reload.

**Notes** — A handful of fields carry help text that documents a convention not recorded anywhere else — most usefully, that the older and newer game generations derive the environment colour from different sources. The recipe carries those in [`Environment_misc.cpp`](Environment_misc.cpp.md) rather than here.

## `show_mixer_parameters`

**Contract** — Draws the *blended* environment's derived fields — the packed environment colour, the blend weight, the modifier attenuation, and the two derived fog planes — on top of the frame form. These are read-outs of the blend, and editing them is overwritten on the next frame; they exist to let an author see why a scene looks the way it does.

## `on_tool_frame`

**Contract** — The editor's panel. Offers: pausing the simulation (so the clock stops and a frame can be edited without the blend moving under the author); reloading the whole weather set from disk, discarding edits; saving the whole set back; selecting the active cycle, the active effect and the two bracketing frames by hand; and editing both bracketing frames side by side with the blend between them shown live.

**Notes** — Manual selection of the bracketing pair is what makes the editor usable without a level loaded: normally the pair is chosen by the clock, and with no level the clock does not run. Selecting a pair by hand drives the same blend the game would.

**Notes** — The pause it offers is the device's own pause, with both the clock and audio suppressed, tagged with a reason string. Reason strings on pause exist because several subsystems pause independently and the engine must know which of them still wants the game stopped.

## `show_level_weathers`

**Contract** — A placeholder panel with two empty groups for single-player and multiplayer level weather assignment. Not implemented.

**Notes** — Nothing to recover. A rebuild either implements per-level weather assignment in the editor or drops the panel.
