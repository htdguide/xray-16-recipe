# src/xrGame/saved_game_wrapper_script.cpp

> Exports the save-file peek to the script virtual machine, so the load screen can be written in Lua.

**Needs** — [`saved_game_wrapper.h`](saved_game_wrapper.h.md) · [`ai_space.h`](ai_space.h.md) · [`xr_time.h`](xr_time.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

The load and save menus are script-driven, so the four facts extracted in
[`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md) must be reachable from Lua. This is the
whole of that surface.

## State

`Stateless.`

## `script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up:

- **the wrapper class**, constructible from a save name, with the four reads — level
  identifier, level name and actor health passed straight through, and the clock **wrapped in
  the script-side time type** rather than exposed as a raw number. That wrap is the only
  translation in the file, and it exists because Lua scripts format the save's date and need
  the calendar operations the time type provides, not an integer.
- **a free function** exposing the name form of the validity check, so a script can ask
  whether a save is loadable before constructing a wrapper over it.

**Notes** — constructing the wrapper from script decompresses an entire save. A script that
builds one per row while drawing a save list is doing real work per frame; the shipped menus
build them once when the list is populated. A rebuild exposing this surface should keep that
cost visible in the name or cache the result.
