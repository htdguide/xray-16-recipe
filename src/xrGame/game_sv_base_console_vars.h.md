# src/xrGame/game_sv_base_console_vars.h

> Empty. A header that once declared the server's console variables and now declares nothing.

**Needs** — _(none)_
**Used by** — [`game_sv_base.h`](game_sv_base.h.md)
**Tier floor** — T4: nothing to write

## Purpose

The file contains only its include guard. It is still included by
[`game_sv_base.h`](game_sv_base.h.md) and still listed in the build description, so it is a
real file with no content.

Its name says what it held: the declarations of the server rules' console variables, which
now live beside their definitions in [`game_sv_base.cpp`](game_sv_base.cpp.md). A rebuild
writes nothing here and deletes the inclusion.

## State

`Stateless.`
