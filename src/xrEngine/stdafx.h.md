# src/xrEngine/stdafx.h

> The prelude every engine translation unit opens with, and the one place that fixes the order those layers must appear in.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`Engine.h`](Engine.h.md) · [`defines.h`](defines.h.md) · [`device.h`](device.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [`xrCDB/xrXRC.h`](../xrCDB/xrXRC.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4: nothing here computes; it is a build-time aggregation that a language with modules expresses as per-file imports, or deletes.

## Purpose

The engine chapter needs seven unrelated vocabularies in scope before any of its sources
mean anything: the platform prelude, the core runtime (filesystem, strings, logging), the
engine's own global handles, the device, the collision database's query cursor, and the
audio interface. This file names them in the one order that compiles, so no engine source
has to remember it.

It also decides two build-shape questions that are properly *configuration*, not code: the
debug-overlay editor is compiled in unless the build says otherwise, and on Windows the
overlay is allowed to detach windows from the main one.

## State

Stateless.

## Notes

A rebuild has no equivalent of this file. The aggregation is an artifact of a language
where every file re-parses its dependencies; the ordering constraint it encodes disappears
the moment imports are resolved by name rather than by textual inclusion.

One real fact hides here: the engine reaches a second configuration file — the
*game*-level one, distinct from the system one — as a process-wide handle. That the game
configuration is a global rather than a parameter is a decision the object registry and the
console both depend on.
