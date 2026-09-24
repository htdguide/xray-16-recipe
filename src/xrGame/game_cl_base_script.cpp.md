# src/xrGame/game_cl_base_script.cpp

> Exports the minimap marker and the respawn point to Lua, so a script-written game mode can draw its own map and place its own spawns.

**Needs** — [`game_cl_base.h`](game_cl_base.h.md) · [`game_base.h`](game_base.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Two tiny registrations that between them let a script contribute to two engine-drawn things:
the minimap and the respawn table.

## State

`Stateless.`

## `SZoneMapEntityData::script_register`

**Contract** — registers the minimap marker (position and packed colour, both read/write,
default-constructible) **and the list type it is collected into**, exposing only an append.

**Notes** — registering the container as a named type with a single append is the whole
mechanism: the engine hands a script the list it is about to draw, the script appends
markers, and the engine draws them. A rebuild whose binding layer can pass a native list to
script does not need the wrapper type, but does need the same "the engine owns the list, the
script fills it" contract — a script must not be able to replace or reorder it.

## `RPoint::script_register`

**Contract** — registers a respawn point exposing only its position and orientation, both
read/write. The reservation fields — who currently holds this point and since when — are
deliberately not exposed: reservation is the server's arbitration and a script that could
write it would break the guarantee that two players never materialize in the same place.
