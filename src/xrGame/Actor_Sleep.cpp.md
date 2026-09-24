# src/xrGame/Actor_Sleep.cpp

> Empty.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: nothing to implement.

## Purpose

A file that once held the actor's sleep behaviour and now holds nothing but the shared
prelude. The behaviour it named moved into the script layer: sleeping is a script-driven
sequence that advances the game clock by the configured sleep duration
([`Actor_Flags.h`](Actor_Flags.h.md)) and runs a fade effector, and nothing in the engine
needs a C-side entry point for it.

A rebuild should not recreate this file. It is listed here only so the mirror is complete.
