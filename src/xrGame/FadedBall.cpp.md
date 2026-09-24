# src/xrGame/FadedBall.cpp

> Another effectless artefact class: identity only, behaviour entirely inherited.

**Needs** — [`FadedBall.h`](FadedBall.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identity and nothing else

## Purpose

A shipped artefact whose class adds nothing over the base artefact — its immunities,
condition effects, mass and visual all come from its configuration section. The class
exists because the class identifier in the shipped spawn data must resolve to something.

Several artefacts in this directory are exactly this shape
([`DummyArtifact.cpp`](DummyArtifact.cpp.md) is the other in this slice). A rebuild that
drives artefact effects from data can collapse them all into one class selected by
section, keeping only the identifier-to-behaviour mapping.

## State

`Stateless.`

## `CFadedBall`

**Contract** — construction, destruction and configuration load delegate to the base
artefact.
