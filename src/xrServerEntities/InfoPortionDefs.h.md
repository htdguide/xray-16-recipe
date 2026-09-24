# src/xrServerEntities/InfoPortionDefs.h

> One record: a piece of knowledge a character has, and when they learned it.

**Needs** — [`alife_space.h`](alife_space.h.md) · [`Common/object_interfaces.h`](../Common/object_interfaces.h.md)
**Used by** — [`InfoDocument.cpp`](../xrGame/InfoDocument.cpp.md) · [`InfoDocument.h`](../xrGame/InfoDocument.h.md) · [`InfoPortion.cpp`](../xrGame/InfoPortion.cpp.md) · [`InventoryOwner.h`](../xrGame/InventoryOwner.h.md) · [`PDA.h`](../xrGame/PDA.h.md) · [`PhraseScript.h`](../xrGame/PhraseScript.h.md) · [`alife_registry_container_composition.h`](../xrGame/alife_registry_container_composition.h.md) · [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md)
**Tier floor** — T1: the pair is serialized into character records at fixed widths.

## Purpose

The game's entire quest and dialogue state is a set of named facts — *info portions* — that
characters either know or do not. A conversation branch, a task's completion, a faction's
opinion and a map marker are all expressed as "does this character have this info". This
file is the record of one such fact on one character.

## State

```text
RECORD KnownInfo
  info_id      : text              # the authored name of the fact
  receive_time : int (64-bit)      # game time, milliseconds, when it was learned
```

**Invariants**

- The identifier is an authored name from the info-portion XML, shared rather than copied —
  the same few hundred names appear across every character in the world, so they are
  interned.
- The time is the alife game clock, not real time, so a save restores it meaningfully.
- A character's known-info list holds each identifier at most once. Lookup is a linear scan
  by identifier; the lists are short (tens of entries) and are searched rarely, so nothing
  more is warranted.

## `load` / `save`

**Contract** — the pair is written as its two fields in order, identifier then time, through
the project's generic serialization. Defined in the game module rather than here, which is a
file-placement accident; the record and its serialization are one idea.

**Notes** — the receive time is carried for exactly one reason: some dialogue and some
scripts ask *how long ago* a fact was learned, not merely whether it is known. Without it
the list would be a plain set of names.
