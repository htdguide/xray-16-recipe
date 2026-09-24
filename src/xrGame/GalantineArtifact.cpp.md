# src/xrGame/GalantineArtifact.cpp

> An artefact kind distinguished only by its class identifier and its configuration section; it adds no behaviour to the base artefact.

**Needs** — [`GalantineArtifact.h`](GalantineArtifact.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a named leaf of the entity class hierarchy

## Purpose

Artefacts differ from one another almost entirely through their configuration section —
the effects they apply to a carrier, their visual and their physics parameters are all
tuned numbers, not code. A few artefact kinds nevertheless need their own class
identifier because the spawn data names one, and this is such a kind (the "witch's
jelly"). It exists so the factory has a constructor to call.

## State

`Stateless.` — everything is inherited from the base artefact.

## `CGalantineArtefact`

**Contract** — loading from a section delegates unchanged to the base artefact's load.
No behaviour is added anywhere in the entity lifecycle.

**Notes** — the explicit delegating override is an artifact of a language where
overriding is opt-in per method; a rebuild simply does not declare the class, or declares
it as an alias, and keeps the class-identifier → section mapping in data.
