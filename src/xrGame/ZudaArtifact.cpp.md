# src/xrGame/ZudaArtifact.cpp

> The "zuda" artefact: an artefact leaf with no behaviour beyond its class identifier and its configuration section.

**Needs** — [`ZudaArtifact.h`](ZudaArtifact.h.md) · [`Artefact.h`](Artefact.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md)
**Used by** — reached through its declarations in [`ZudaArtifact.h`](ZudaArtifact.h.md); callers name that, not this file.
**Tier floor** — T3: pure delegation

## Purpose

One of the artefact leaves that exists only so the class-identifier factory has a distinct
constructor to call. Its load path forwards straight to the artefact base; every property
that distinguishes it — its carry effects, its immunities, its model, its price — lives in
its configuration section.

Grouping artefacts by class the way this directory does is historical: some artefact
leaves (the hovering one, the shocking one) really do have behaviour, and the rest were
given classes to match. A rebuild should keep classes only for the leaves that override
something.

## State

`Stateless.`

## `CZudaArtefact`

**Contract** — an artefact with no overridden behaviour. Its `Load` forwards to the base
unchanged.
