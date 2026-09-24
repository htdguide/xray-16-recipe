# src/xrGame/ThornArtifact.cpp

> The "thorn" artefact: a named artefact type with no behaviour of its own.

**Needs** — [`ThornArtifact.h`](ThornArtifact.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identifier with a name

## Purpose

Like [`RustyHairArtifact.cpp`](RustyHairArtifact.cpp.md), an artefact whose hand-written
behaviour was removed: all of its effect is configuration on the generic artefact. The
class exists so the shipped class-identifier table has something to construct and so
scripts can identify the type.

## State

`Stateless.`

## `CThornArtefact`

**Contract** — an artefact. `Load` defers entirely to the generic artefact. Construction
and destruction do nothing.

**Notes** — the file still pulls in the physics-shell interface, which nothing in it uses.
A leftover of the removed behaviour, and a hint that the thorn once pushed things around.
