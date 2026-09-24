# src/xrGame/artefact_script.cpp

> Exports the artefact base class and every shipped artefact type to the script virtual machine.

**Needs** — [`Artefact.h`](Artefact.h.md) · [`MercuryBall.h`](MercuryBall.h.md) · [`GraviArtifact.h`](GraviArtifact.h.md) · [`BlackDrops.h`](BlackDrops.h.md) · [`BastArtifact.h`](BastArtifact.h.md) · [`BlackGraviArtifact.h`](BlackGraviArtifact.h.md) · [`DummyArtifact.h`](DummyArtifact.h.md) · [`ZudaArtifact.h`](ZudaArtifact.h.md) · [`ThornArtifact.h`](ThornArtifact.h.md) · [`FadedBall.h`](FadedBall.h.md) · [`ElectricBall.h`](ElectricBall.h.md) · [`RustyHairArtifact.h`](RustyHairArtifact.h.md) · [`GalantineArtifact.h`](GalantineArtifact.h.md) · [`Needles.h`](Needles.h.md) · [`cta_game_artefact.h`](cta_game_artefact.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point declares the artefact hierarchy to Lua. Its shape is the point
of the file: the base artefact carries the entire script surface, and the twelve concrete
artefact types are registered as *names with a constructor and nothing else*. A script can
ask whether an object is a mercury ball, but there is nothing a mercury ball can do that an
artefact cannot.

## State

`Stateless.`

## `CArtefact::script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up.

- **`CArtefact`**, deriving from the game object, with a default constructor and three
  methods: put the artefact on its authored flight path, toggle its visibility, and read its
  rank. Rank is what the trade and detector systems sort by.
- **Twelve concrete artefacts**, each deriving from `CArtefact` with a default constructor
  and no methods of their own: mercury ball, black drops, black gravitational artefact,
  bast, dummy, zuda, thorn, faded ball, electric ball, rusty hair, galantine, and
  gravitational artefact.

Names and signatures are frozen by conformance criterion 10.

**Notes** — the needle artefact and the capture-the-artefact multiplayer artefact are pulled
in as dependencies but never registered. The first is an omission; the second is registered
by the multiplayer game mode that owns it.

**Notes** — the default constructors exist because the binding layer requires a constructible
type to derive a script class from one, not because scripts ever construct an artefact.
Entities are created by the spawn factory from a class identifier; a script-constructed
artefact would have no server record.
