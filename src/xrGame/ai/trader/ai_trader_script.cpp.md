# src/xrGame/ai/trader/ai_trader_script.cpp

> Exports the trader's class identity to the script layer.

**Needs** — [`ai_trader.h`](ai_trader.h.md) · [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration

## Purpose

Registers the trader as a script-visible class deriving from the generic game object, with a
default constructor and no methods of its own.

The emptiness is more pointed here than for a creature. The trader has *no* behaviour except
what scripts give it, and yet the script layer reaches it entirely through the generic
game-object facade — the action queue, the dialogue manager, the inventory and the trade
callbacks are all exported there, not here. So a rebuild must keep the trader's script
surface identical without any of it living in this file.

## `script_register`

**Contract** — declares the class, its script-visible name, its base class and a default
constructor into the script virtual machine. Called once at script-layer start-up. The
exported name is frozen by conformance criterion 10.
