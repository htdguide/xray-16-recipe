# src/xrPhysics/tri-colliderknoopc/dTriCallideK.cpp

> Forces the three primitive-versus-triangle headers to be compiled once.

**Needs** — [`dTriCollideK.h`](dTriCollideK.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: it exists to make a compiler emit something.

## Purpose

A translation unit whose only content is the umbrella include. Its job is to give the
inline-heavy primitive headers one place to be instantiated, so that anything they define
out-of-line exists exactly once in the program.

Entirely incidental: it is a build artefact of a header-only design, and a rebuild in a
language with modules or a single compilation model has no successor for it. (The misspelling
in the file name is in the original.)

## Stateless.
