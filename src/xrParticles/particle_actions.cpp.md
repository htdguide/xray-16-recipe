# src/xrParticles/particle_actions.cpp

> An empty translation unit: the action base type and the action list are entirely inline.

**Needs** — [`particle_actions.h`](particle_actions.h.md)
**Used by** — reached through its declarations in [`particle_actions.h`](particle_actions.h.md); callers name that, not this file.
**Tier floor** — T4: a build-system artifact with no runtime meaning.

## Purpose

Stateless, and empty of behaviour. Everything the action list is lives in
[`particle_actions.h`](particle_actions.h.md); this file exists only so that the header has a
compiled companion in the build graph. A rebuild deletes it.
