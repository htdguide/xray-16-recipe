# src/xrGame/UIGame_custom_script.cpp

> Lets a script define its own game UI layer by subclassing the engine's.

**Needs** — [`UIGame_custom_script.h`](UIGame_custom_script.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data plus one dispatch decision

## Purpose

Exports the game-UI base to the script virtual machine as a type scripts may *derive
from*, not merely call. That is the whole point of the file: a mod can supply its own
heads-up layer entirely in script.

## State

`Stateless.`

## `UIGame_custom_script::script_register`

**Contract** — registers a type named `UIGame_custom_script`, deriving from the engine's
game-UI facade, default-constructible from script, exposing two overridable entry points:
bring-up (`Init`) and "here is the client game object you belong to" (`SetClGame`).

**Invariants** — each of the two is registered *twice*: once as the engine implementation
and once as the dispatch that prefers a script override. The consequence a rebuild must
reproduce: calling either from the engine runs the script's version when the script
defined one, and the engine's version otherwise, and a script override can still reach
the engine version explicitly. Both names are frozen by conformance criterion 10.

**Notes** — the override dispatch is expressed in C++ as a wrapper template; the decision
underneath is "this exported type supports script-side inheritance with virtual
dispatch back into script", which any binding layer expresses its own way.
