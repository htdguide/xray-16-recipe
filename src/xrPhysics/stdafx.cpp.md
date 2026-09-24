# src/xrPhysics/stdafx.cpp

> The translation unit that materializes the precompiled header.

**Needs** — [`StdAfx.h`](StdAfx.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: a build artifact with no runtime meaning.

## Purpose

Entirely incidental: it exists so the toolchain has one compilation unit from which to build the
shared precompiled header. It contributes no code, no state and no decisions, and has no
counterpart in a rebuild.
