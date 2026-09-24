# src/xrParticles/stdafx.cpp

> The translation unit that exists so the module's precompiled prelude has something to compile into.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.
**Tier floor** — T4: a build-system artifact with no runtime meaning.

## Purpose

Stateless. It contributes no code, no data and no behaviour. A rebuild deletes it.
