# src/xrGame/UIGame_custom_script.h

> Declares the script-derivable game UI registered in [`UIGame_custom_script.cpp`](UIGame_custom_script.cpp.md).

**Needs** — [`UIGameCustom.h`](UIGameCustom.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`UIGame_custom_script.cpp`](UIGame_custom_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UIGame_custom_script`, a game-UI subclass whose two entry points do nothing so
that a script subclass can supply them. Substance — the registration and its dispatch
rules — is in [`UIGame_custom_script.cpp`](UIGame_custom_script.cpp.md).

Exported units:

- `UIGame_custom_script` — an empty game UI intended to be subclassed from script.
- `Init` — bring-up; the engine version does nothing.
- `SetClGame` — hand over the client game object; the engine version just forwards to the
  base.
