# src/Common/GUID.hpp

> A 128-bit identity stamped into level build outputs so that files produced by one build of a level can be checked to belong together.

**Needs** — [`xrCore/xrCore.h`](../xrCore/xrCore.h.md)
**Used by** — [`LevelStructure.hpp`](LevelStructure.hpp.md) · [`game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Tier floor** — T1: it is a fixed 16-byte field embedded in a memory-image file header, so its layout is frozen.

## Purpose

A level is compiled into several files by separate tools — geometry, collision, the AI
navigation mesh, the spawn list. If a rebuild of one is paired with a stale copy of another
the mismatch is silent and the symptoms are bizarre, so the compiler stamps every output of
one run with the same identity and the loaders compare them.

## State

```text
RECORD Guid
  halves : list<int (64-bit)>   # exactly two; 16 bytes, little-endian, written as a
                                # memory image into level file headers
```

**Invariants** — the value is opaque: nothing reads a field of it, only compares whole values
for equality. Two identities are equal when both halves are equal. The byte layout is
frozen by [the level data format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence).

## `equals` / `not_equals`

**Contract** — whole-value comparison over both halves. No ordering is defined, because nothing
sorts identities.

## `load_from_config` / `save_to_config`

**Contract** — read or write the identity in the text configuration format, under a caller-given
section and a caller-given base name. The two halves are stored as two separate unsigned
64-bit decimal entries, named by appending `_g0` and `_g1` to the base name.

```text
FUNCTION load_from_config(config, section, base_name) -> Guid
  halves[0] <- config.read_u64(section, base_name + "_g0")
  halves[1] <- config.read_u64(section, base_name + "_g1")

FUNCTION save_to_config(config, section, base_name, guid)
  config.write_u64(section, base_name + "_g0", guid.halves[0])
  config.write_u64(section, base_name + "_g1", guid.halves[1])
```

**Notes** — the split into two decimal entries exists because the configuration format has no
128-bit scalar and no byte-string literal. The suffix spellings are load-bearing: tool
output already on disk uses them.

This is *not* a universally-unique identifier in the standardized sense — there is no
version or variant nibble, and nothing in this engine generates one from a clock or a
network address. It is sixteen opaque bytes chosen by whatever produced the level.
