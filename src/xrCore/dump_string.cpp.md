# src/xrCore/dump_string.cpp

> Human-readable renderings of the math types, for debugging — present only in development builds.

**Needs** — [`dump_string.h`](dump_string.h.md) · [`_fbox.h`](_fbox.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_matrix.h`](_matrix.h.md) · [`log.h`](log.h.md) · [`xrDebug.h`](xrDebug.h.md)
**Used by** — [`dump_string.h`](dump_string.h.md)
**Tier floor** — T3: formatting.

## Purpose

When a physics or animation bug is being chased, the fastest thing is to print a transform and read it. This file is those printings, and the whole file is absent from shipping builds.

## `get_string`

**Contract** — Renders a value as text, returning an owning string. One overload per type:

| Type | Rendering |
|---|---|
| bool | `true` / `false` |
| 3-vector | `( x, y, z )` |
| 4x4 matrix | four newline-separated rows of four components, first axis, second axis, third axis, translation |
| axis-aligned box | `[ min: (…) - max: (…) ]` |

**Invariants** — The matrix rows are the three basis axes and the translation, each followed by that row's fourth component. That ordering is the engine's matrix convention and the rendering is the clearest statement of it in the codebase: **a transform is three basis vectors and a translation, stored row-wise.**

## `dump_string` and `dump`

**Contract** — The same renderings prefixed by a caller-supplied name; `dump` logs the result rather than returning it. The matrix form names each row after the parent name — `name.i`, `name.j`, `name.k`, `name.c` — so a log line identifies which transform and which row.

**Notes** — Building the matrix rendering by nested formatting is wasteful and irrelevant; it runs when a developer asks it to.
