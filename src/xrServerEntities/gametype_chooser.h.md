# src/xrServerEntities/gametype_chooser.h

> Which game modes an authored entity exists in — a 16-bit mask stored on every spawn record — plus the mode identifiers themselves and the parse from their configuration spellings.

**Needs** — [`gametype_chooser.cpp`](gametype_chooser.cpp.md) · [`xrCore/_flags.h`](../xrCore/_flags.h.md)
**Used by** — [`UIGameCustom.h`](../xrGame/UIGameCustom.h.md) · [`game_base.cpp`](../xrGame/game_base.cpp.md) · [`game_base.h`](../xrGame/game_base.h.md) · [`UIMapList.h`](../xrGame/ui/UIMapList.h.md) · [`UIOptConCom.cpp`](../xrGame/ui/UIOptConCom.cpp.md) · [`GameSpy_Browser.cpp`](../xrGameSpy/GameSpy_Browser.cpp.md) · [`PropertiesListTypes.h`](PropertiesListTypes.h.md) · [`gametype_chooser.cpp`](gametype_chooser.cpp.md) · [`xrEProps.h`](xrEProps.h.md) · [`xrServer_Object_Base.cpp`](xrServer_Object_Base.cpp.md) · [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md)
**Tier floor** — T1: the mask is a 16-bit field in every spawn record.

## Purpose

One level file serves every game mode. A respawn point that exists only in capture-the-
artefact, an artefact that exists only in artefact hunt, a door that exists only in single
player — all are in the same spawn file, each carrying a mask of the modes it participates
in. The loader tests the mask against the running mode and skips what does not belong.

This file also owns the mode identifiers, which are frozen twice over: they are in the mask
bit positions on disk, and they are in the server list protocol that the three games'
different vintages share.

## State

```text
ENUM GameMode : int (32-bit, used as bit positions)
  none                  = 0
  single                = bit 0
  deathmatch            = bit 1
  team_deathmatch       = bit 2
  artefact_hunt         = bit 3
  capture_the_artefact  = bit 4
  domination_zone       = bit 5
  team_domination_zone  = bit 6

RECORD GameModeMask
  modes : int (16-bit bitfield)      # written to and read from every spawn record
```

**Invariants**

- The mask defaults to **all bits set** — an entity with no explicit mode restriction exists
  in every mode. That is what makes the field free for the overwhelming majority of records.
- The "matches" test answers true when the queried mode is *none*, which is how tools that
  do not care about modes see every entity.
- The mask is 16 bits on disk while the identifiers are declared as a 32-bit enumeration.
  Only the low seven are used, so nothing is lost, but a rebuild must write 16.

## the older numbering

*Shadow of Chernobyl* numbered the modes **densely** — 6 for team deathmatch, 7 for artefact
hunt — where the later games use bit positions. Both numberings reach the server-list
protocol, so a second enumeration records the two old values that collide with the bit
scheme. There is no conversion between them at run time; the old values appear only in the
master-list display path.

## `parse_mode_name`

**Contract** — maps a configuration spelling to a mode identifier, accepting both the long
form and the abbreviation (`deathmatch` or `dm`, `teamdeathmatch` or `tdm`, `artefacthunt`
or `ah`, `capturetheartefact` or `cta`; the two domination modes have long forms only).
An unrecognized name answers *none*. Case-sensitive, because every shipped spelling is
lowercase.

**Notes** — the two domination modes are declared, parseable and stored, and **no code
implements them**. They are cut content whose identifiers survived into the format. A
rebuild keeps the bits so that shipped masks parse, and implements nothing.

## `set_defaults` / `matches`

**Contract** — `set_defaults` sets every bit, which is what a freshly created record gets.
`matches(mode)` is the loader's test described above.

## the editor surface

The record also declares read and write of the mask against a binary stream and against a
configuration section, and its contribution to the property editor. All three are compiled
only into the tools; see [`gametype_chooser.cpp`](gametype_chooser.cpp.md).
