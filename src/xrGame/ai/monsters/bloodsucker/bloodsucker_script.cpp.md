# src/xrGame/ai/monsters/bloodsucker/bloodsucker_script.cpp

> The bloodsucker's script surface: one method.

**Needs** — [`bloodsucker.h`](bloodsucker.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding registration

## Purpose

Registers the bloodsucker as a script-visible class deriving from the game object, with a
default constructor and exactly one exported method: **force the visibility state**, taking
the state as an integer.

The class name and the method name are frozen by conformance criterion 10 — a shipped script
calls them by these names.

## Notes

The forced state is an integer at the script boundary rather than a named enumeration, so a
script passes a bare number: −1 unsets the override, 0 hides the creature, 1 makes it
partially visible, 2 makes it fully visible. Nothing exports those names, so a script author
must know them. See [`bloodsucker.cpp`](bloodsucker.cpp.md).
