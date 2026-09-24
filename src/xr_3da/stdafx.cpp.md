# src/xr_3da/stdafx.cpp

> Exists only so the build system has a file to compile the shared header into; it contributes no behaviour.

**Needs** — [`stdafx.h`](stdafx.h.md)

**Used by** — reached through its declarations in [`stdafx.h`](stdafx.h.md); callers name that, not this file.

**Tier floor** — T4: it has no content.

## Purpose

A build-tool artifact with no counterpart in a rebuild. The original's compiler needed one
translation unit designated as the place where the shared header is compiled once and
cached; this is that unit. Delete it.

## Exported units

None.
