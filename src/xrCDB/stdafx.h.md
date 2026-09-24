# src/xrCDB/stdafx.h

> The module's precompiled header — names the four things every file in the
> directory needs, so the build does not reprocess them per file.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`StdAfx.cpp`](StdAfx.cpp.md)
**Tier floor** — T4: it is a build-time declaration, not code.

## Purpose

Purely a compilation artifact. It has no runtime existence and decides nothing about the
system; it exists because the original's tier reprocesses declarations once per translation
unit and this collapses that cost.

The one fact worth carrying out of it: it is where the vendored tree library is pulled in
for the whole directory. That is the boundary of the delegated part — a rebuilder replacing
the tree builder replaces what this names, and nothing else in the module changes.

## State

Stateless.

## Notes

A rebuild in almost any other tier deletes this file. Where the tier still wants an
aggregation point, the content survives as *this module depends on the core layer and on one
external tree implementation* — which is already stated in
[`README.md`](README.md) and in [`CMakeLists`](../../SYSTEM-REQUIREMENTS.md#7-build-order).
