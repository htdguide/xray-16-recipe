# src/xrGame/alife_simulator_header.cpp

> The save file's version stamp: written first, checked before anything else is read, and the reason a mismatched save is refused rather than guessed at.

**Needs** — [`alife_simulator_header.h`](alife_simulator_header.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — reached through its declarations in [`alife_simulator_header.h`](alife_simulator_header.h.md); callers name that, not this file.
**Tier floor** — T2: a single integer in its own chunk

## Purpose

Four lines of real content, and they implement the policy §5 states: *the engine refuses
mismatched versions rather than guessing*. Every other part of a saved game is written
without a tag, a length or a checksum, and is parseable only because the writer and the
reader agree on the order. This chunk is what establishes that they do.

## State

```text
RECORD SimulatorHeader
  version : int (32-bit)    # ALIFE_VERSION at construction; the file's value after a load
```

The version constant is a single number covering the *entire* alife save format — every
registry, every entity's serialized state, the whole container layout. There is no
per-section version. That is the deliberate simplification, and its cost is that any
format change anywhere invalidates every save.

## `save`

**Contract** — writes one chunk containing the current format version, and nothing else.
Always the first chunk of an alife save.

```text
FUNCTION save(stream)
  open chunk ALIFE_CHUNK_DATA
  stream.write_int32(ALIFE_VERSION)      # the constant, never the loaded value
  close chunk
```

**Invariants** — the *constant* is written, not the version that was loaded. A save
produced by this build always claims this build's version, even when it began as an older
file that was accepted — which is correct, because everything else in the file was just
written in this build's format.

## `load`

**Contract** — reads the version back. Fails hard if the chunk is missing or if the file
is older than this build.

```text
FUNCTION load(stream)
  REQUIRE stream has chunk ALIFE_CHUNK_DATA   ELSE FAIL WITH missing chunk
  version = stream.read_int32()
  REQUIRE version >= ALIFE_VERSION
    ELSE FAIL WITH "version mismatch — delete the saved game"
```

**Invariants** — the test is **at least**, not **equal**. A file claiming a *newer*
version than this build is accepted; only an older one is refused. That asymmetry is
almost certainly not what was meant — a newer file's layout is exactly as unreadable as an
older one's — but it is the shipped behaviour and a rebuild reproducing the original's
tolerance will accept the same set of files. A rebuild that tightens it to equality is
strictly safer and will reject nothing the original could actually read.

The message names the remedy rather than the cause, because the only remedy is to discard
the save.

## `valid`

**Contract** — the same check, without failing: reports whether a stream looks like a
loadable alife save. Used to filter the save list the player is offered, so that an
unloadable file is greyed out rather than crashing the game when picked.

**Notes** — `valid` and `load` apply the same comparison in two different styles — one
reporting, one fatal — and a rebuild should implement one and derive the other, since a
divergence between them means the menu offers a save the loader then refuses.
