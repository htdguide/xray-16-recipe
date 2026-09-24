# src/xrEngine/stdafx.cpp

> The compilation anchor for the chapter's prelude; it contains no program.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: a build artifact with no behaviour.

## Purpose

Exists so the toolchain has one translation unit in which to materialise the shared
prelude. It has no counterpart in a rebuild.

## State

Stateless.
