# src/xrGame/RustyHairArtifact.cpp

> The "rusty hair" artefact: a named artefact type with no behaviour of its own.

**Needs** — [`RustyHairArtifact.h`](RustyHairArtifact.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — reached through its declarations in [`RustyHairArtifact.h`](RustyHairArtifact.h.md); callers name that, not this file.
**Tier floor** — T3: a class identifier with a name

## Purpose

One of the artefacts that once had a hand-written effect and no longer does. Everything it
does — the passive bonuses and penalties it grants while carried, its restoration rates,
its model and its trade value — comes from its configuration section through the generic
artefact. The class adds a name and nothing else.

It survives as a separate class only because the class-identifier table in the shipped
spawn data names it. In a rebuild that resolves entity classes from the section rather
than from a compiled tag, this file and its header disappear.

## State

`Stateless.`

## `CRustyHairArtefact`

**Contract** — an artefact. `Load` reads the section by deferring entirely to the generic
artefact. Construction and destruction do nothing.
