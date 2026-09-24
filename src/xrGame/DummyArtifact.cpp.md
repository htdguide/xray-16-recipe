# src/xrGame/DummyArtifact.cpp

> An artefact with no effect: the base artefact behaviour under its own class identifier.

**Needs** — [`DummyArtifact.h`](DummyArtifact.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identity and nothing else

## Purpose

One of the shipped artefact class identifiers maps to "an artefact that does nothing
beyond what every artefact does" — it can be found, carried, traded and attached to a
suit, and it contributes whatever immunities and condition modifiers its configuration
section declares, but it adds no per-frame behaviour of its own. It exists as a separate
class because the spawn records in the shipped level data name it, and the class
identifier is frozen (see [`SYSTEM-REQUIREMENTS.md` §5](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)).

## State

`Stateless.`

## `CDummyArtefact`

**Contract** — construction, destruction and configuration load are pure delegations to the
base artefact. A rebuild that dispatches artefact behaviour from data rather than from a
class hierarchy needs no type here at all, only the class identifier's entry in the
spawn factory table.
