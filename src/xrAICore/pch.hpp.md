# src/xrAICore/pch.hpp

> The module's shared prelude — the three things every file in the AI core starts from.

**Needs** — [`../Common/Common.hpp`](../Common/Common.hpp.md) · [`../xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [`AISpaceBase.hpp`](AISpaceBase.hpp.md)
**Used by** — [`pch.cpp`](pch.cpp.md)
**Tier floor** — T4: it is a build artefact. Nothing in it is a decision about the system.

## Purpose

A compilation accelerator, not a design element: it names the headers every file in this module
would include anyway, so the compiler parses them once per module rather than once per file.

The only thing a rebuild should take from it is the observation that the AI core rests on exactly
three things — the platform and compiler abstraction, the core runtime (containers, strings,
filesystem, math), and its own AI-space handle through which the graphs are reached. That short
list is a real statement about the module's coupling: the AI core does not depend on the game, the
renderer, or the physics.

## Stateless.

**Notes** — a rebuild in a language with a module system has nothing to write here.
