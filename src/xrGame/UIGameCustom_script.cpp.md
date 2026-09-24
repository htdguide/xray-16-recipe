# src/xrGame/UIGameCustom_script.cpp

> The in-game interface as scripts see it: put a named piece of text on the screen, open or close the inventory and the handheld computer, and reach the whole thing from anywhere.

**Needs** — [`UIGameCustom.h`](UIGameCustom.h.md) · [`Level.h`](Level.h.md) · [`xrUICore/Static/UIStatic.h`](../xrUICore/Static/UIStatic.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one declarative registration block

## Purpose

Exports the in-game interface to the script layer. Frozen by conformance criterion 10 — the
games' own scripts and every modification's own screens address these names.

The surface is deliberately narrow. A script may **add its own screens** (through the
inherited dialog-holder operations), **show text overlays**, and **open and close the two
built-in screens**. It may not touch the heads-up display's own panels, which is why so much
of a modification's user interface is built out of custom screens rather than by rearranging
the shipped ones.

## State

`Stateless.`

## `script_register`

**Contract** — register two classes and one free function with the script virtual machine.
Runs once at startup.

Exported as `StaticDrawableWrapper` — one text overlay:

- `m_endTime` — readable *and writable*: a script extends or shortens an overlay's life by
  assigning it. This is the only field of any interface object a script may write directly.
- `wnd` — the underlying text widget, through which the script sets font, colour, position
  and text. Handing out the widget rather than wrapping it is what makes the overlay
  system useful and is also why the widget's own surface is effectively frozen too.

Exported as `CUIGameCustom`, declared as deriving from the dialog holder:

- `AddDialogToRender`, `RemoveDialogToRender` — inherited: this is how a script-built screen
  gets drawn at all.
- `AddCustomStatic` — **exported twice**, once with the time-to-live argument and once
  without. The two-argument form is a wrapper that calls the three-argument one with the
  default, because the binding layer resolves overloads by argument count at call time and
  cannot see a default argument. A rebuild whose binding understands defaults exports one.
- `GetCustomStatic`, `RemoveCustomStatic` — find and drop an overlay by name.
- `ShowActorMenu`, `HideActorMenu`, `UpdateActorMenu`, `CurrentItemAtCell` — the inventory
  screen. The last answers "which item is the cursor over", which is what every
  modification's custom tooltip and context menu is built on.
- `HidePdaMenu` — the handheld computer can be *closed* from a script but **not opened**:
  `ShowPdaMenu` exists and is not exported. Whether that is deliberate is not recoverable.
- `show_messages`, `hide_messages` — the message log, under lowercase names while everything
  around them is capitalized. Both spellings ship; neither may be changed.
- `update_fake_indicators`, `enable_fake_indicators` — override the condition readouts, so a
  scripted sequence can show the player a value their character does not have.

Exported as a free function:

- `get_hud` — the process-wide accessor, returning the current in-game interface or nothing
  when no level is loaded. A script calling any of the above without checking this first
  is the most common way a modification crashes on the main menu.

**Notes** — the mixed naming (`AddCustomStatic` beside `show_messages`) is the fossil record
of two eras of the codebase, and both are load-bearing because both appear in shipped
scripts. There is no correct set to converge on; export what is here.
