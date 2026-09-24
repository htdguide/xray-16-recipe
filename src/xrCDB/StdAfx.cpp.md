# src/xrCDB/StdAfx.cpp

> The translation unit that exists only so the precompiled header has something to
> be generated from.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — reached through its declarations in [`StdAfx.h`](StdAfx.h.md); callers name that, not this file.
**Tier floor** — T4: build scaffolding.

## Purpose

One line of source. It decides nothing, contains nothing, and exists because the original's
toolchain needs a compilation unit to anchor a precompiled header to.

## State

Stateless.

## Notes

Delete it in a rebuild. Nothing is lost.
