# src/xrMaterialSystem/stdafx.cpp

> The compilation unit that exists so the module's prelude has something to be pre-compiled from.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.
**Tier floor** — T4: a build artifact with no content.

## Purpose

Empty apart from including the prelude. One toolchain requires a translation unit to
anchor a pre-compiled header to; this is that anchor.

## State

Stateless.

## Notes

Delete it. There is nothing here to rebuild.
