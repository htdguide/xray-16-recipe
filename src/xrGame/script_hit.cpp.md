# src/xrGame/script_hit.cpp

> Nothing: the script hit is entirely inline.

**Needs** — [`script_hit.h`](script_hit.h.md)
**Used by** — reached through its declarations in [`script_hit.h`](script_hit.h.md); callers name that, not this file.
**Tier floor** — T2

## Purpose

An otherwise empty translation unit holding the type's destructor so the compiler has one
place to emit its shared parts. The substance is in
[`script_hit.h`](script_hit.h.md) and
[`script_hit_inline.h`](script_hit_inline.h.md). A rebuild has no equivalent of this file.
