# src/Layers/xrAPI/stdafx.cpp

> The translation unit that exists so the module's precompiled prelude has something to compile into.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: a build-system artifact with no runtime meaning.

## Purpose

Stateless. It contributes no code, no data and no behaviour; it is the anchor a particular
compiler toolchain requires in order to build a precompiled header. A rebuild deletes it.
