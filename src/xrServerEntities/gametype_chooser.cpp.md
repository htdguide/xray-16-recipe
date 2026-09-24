# src/xrServerEntities/gametype_chooser.cpp

> Reads the game-mode mask from both format generations, and exposes it to the editor as one property.

**Needs** — [`gametype_chooser.h`](gametype_chooser.h.md) · [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) · [`xrEProps.h`](xrEProps.h.md)
**Used by** — [`gametype_chooser.h`](gametype_chooser.h.md)
**Tier floor** — T1: the older format's value is a one-byte dense index that must be widened into a bit mask.

## Purpose

The whole substance of this file is one conversion: **the first game stored a single mode
number, the later ones store a mask**, and a level authored under the old scheme must still
load. Everything else here is editor plumbing.

## State

```text
ENUM LegacyGameMode : int (8-bit)      # the Shadow of Chernobyl spawn format
  any = 0, deathmatch = 1, team_deathmatch = 2,
  artefact_hunt = 3, capture_the_artefact = 4
```

**Invariants** — the legacy values are dense and unrelated to the bit positions in
[`gametype_chooser.h`](gametype_chooser.h.md). The mapping is a table, not arithmetic.

## `read_from_configuration`

**Contract** — reads the mask from a configuration section, choosing by a flag which
generation of the format the section is in.

```text
FUNCTION read_mask(section, is_legacy : bool)
  IF is_legacy
    n = section."game_type" as one byte
    mask = empty
    IF n = any                  THEN mask = every bit set
    ELSE IF n names a mode      THEN mask = that mode's bit
    # an unrecognized legacy value leaves the mask empty: the entity exists in no mode
  ELSE
    mask = section."game_type" as sixteen bits
```

**Notes** — the legacy flag is the caller's, derived from the spawn file's own version
number (the boundary is spawn format revision 0x14). It is not inferable from the value,
because a dense 3 and a mask with bit 1 set are the same byte. A rebuild must therefore
thread the version through to this read, which is the general pattern across this whole
chapter.

The "any" case expands to *every* bit, including the two unimplemented domination modes.
That is correct: "any" meant any, and the mask is only ever tested against a mode that
actually runs.

## `read_from_stream` / `write_to_stream` / `write_to_configuration`

**Contract** — the binary form is one 16-bit value and nothing else; the configuration form
is the same value under the key `game_type`. There is no legacy branch on the write side —
the tools only ever write the current format.

## `fill_properties`

**Contract** — adds the mask to an entity's property list as a single game-type property,
which the editor renders as a set of checkboxes. Compiled out of the shipping build.

**Notes** — the source keeps, commented out, an earlier version that published seven
separate flag properties, one per mode. It was replaced by one composite property so that
the editor could show a mode picker rather than seven unrelated checkboxes. The behaviour is
identical; only the presentation changed.

The legacy mode-name table (`"Any game"`, `"Deathmatch"`, …) is defined here and used by the
editor's older respawn-point property. It is display text for the dense numbering above.
