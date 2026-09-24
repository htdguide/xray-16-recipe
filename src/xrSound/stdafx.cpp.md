# src/xrSound/stdafx.cpp

> Build scaffolding: the precompiled header's translation unit.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.
**Tier floor** — T4: it decides nothing.

## Purpose

Exists so the compiler has a translation unit in which to build the precompiled header. Pure C++
ecosystem artefact with no counterpart in a rebuild.

## Notes

It additionally defines the vendor reverb extension's interface identifiers, which must be emitted
in exactly one translation unit — see [`guids.cpp`](guids.cpp.md), which does the same job for the
same reason. Having two files both claim it is an inconsistency in the original, tolerated because
the second is excluded from precompilation.
