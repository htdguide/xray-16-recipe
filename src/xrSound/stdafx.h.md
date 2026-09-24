# src/xrSound/stdafx.h

> Build scaffolding: the chapter's shared include set.

**Needs** — [`Sound.h`](Sound.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md)
**Used by** — [`guids.cpp`](guids.cpp.md) · [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4: it decides nothing.

## Purpose

A precompiled-header aggregate. It exists because C++ recompiles every header in every translation
unit; a rebuild in a language with a real module system has no equivalent and should delete it.

The one fact worth extracting: this chapter rests on the platform layer, the core (virtual
filesystem, strings, memory, threading), the reference-counted resource base, and **the static
collision database** — which is the dependency a reader might not expect from an audio module. It
is there because occlusion and reverb regions are both ray queries against level geometry.
