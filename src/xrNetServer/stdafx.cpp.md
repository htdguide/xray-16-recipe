# src/xrNetServer/stdafx.cpp

> The compilation unit that exists so the shared prelude has somewhere to be compiled.

**Needs** — [`stdafx.h`](stdafx.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: build plumbing.

## Purpose

Nothing in the system depends on this file. It exists because the host language's
precompiled-header mechanism needs one translation unit that does nothing but include the
prelude, and because that same unit is where the transport library's identifier constants are
asked to be *defined* rather than merely declared.

A rebuild has neither problem and deletes the file.

## State

Stateless.

## Notes

Both this file and [`guids.cpp`](guids.cpp.md) request the transport library's identifier
definitions in the same way, which means the request is made twice. It is harmless — the
second is a no-op — and it is the kind of duplication that survives because nothing ever
forces anyone to look at either file.
