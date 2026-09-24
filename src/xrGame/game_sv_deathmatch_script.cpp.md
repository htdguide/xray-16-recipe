# src/xrGame/game_sv_deathmatch_script.cpp

> Exports the server-side deathmatch rules to Lua as a subclassable type.

**Needs** — [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`xrServerEntities/xrServer_script_macroses.h`](../xrServerEntities/xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Registers deathmatch's server half, deriving from the registered server-rules base, with a
constructor so a script can instantiate or subclass it.

## State

`Stateless.`

## `game_sv_Deathmatch::script_register`

**Contract** — registers the class with two members: the team-data accessor, and the mode's
name — the latter as an overridable, so a script mode deriving from deathmatch can report a
different name while inheriting all of deathmatch's rules. That is the entire override
surface.

**Notes** — the starting-money setter is registered out, with its wrapper declaration
commented out alongside. Starting money is therefore settable only through the mode's option
string, not from script. If a rebuild wants script-settable economy it is adding a feature,
not restoring one.

The name is the only overridable here, in contrast with the client side's nine (see
[`game_cl_mp_script.cpp`](game_cl_mp_script.cpp.md)). The asymmetry is real: the server's
rules are meant to be replaced wholesale by a script mode deriving from the *base*, not
tweaked by deriving from a shipped mode.
