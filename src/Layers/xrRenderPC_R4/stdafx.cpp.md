# src/Layers/xrRenderPC_R4/stdafx.cpp

> The translation unit that exists so the module's shared header manifest is compiled once.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.
**Tier floor** — T4: it contains no decisions at all.

## Purpose

Purely a build artifact of C++'s precompiled-header mechanism: a file whose only content is the manifest include, so the compiler has somewhere to emit the shared header's compiled form. No behaviour, no state, no decision.

A rebuild in any language with a module system deletes this file and does not replace it with anything.

## State

`Stateless.`
