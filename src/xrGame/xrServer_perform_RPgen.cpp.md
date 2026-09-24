# src/xrGame/xrServer_perform_RPgen.cpp

> Where an entity's spawn position would have been chosen from the level's respawn points — abandoned, and now always a pass.

**Needs** — [`xrServer.h`](xrServer.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: returns success

## Purpose

An entity's spawn record can name *where* to appear in three ways: at the coordinates already in
the record, at a specific numbered respawn point of its team, or at one chosen by the server.
This file was the chooser. It now accepts every spawn unconditionally, and the placement is
decided by the game mode instead.

The page exists because the abandoned code states a design a rebuild has to decide about anyway.

## State

`Stateless.`

## `PerformRP`

**Contract** — validate and, if needed, place a spawning entity. **Always succeeds, changing
nothing.**

The scheme it was written for:

```text
FUNCTION place_at_respawn_point(entity) -> bool
  points := respawn points of entity's team
  IF entity's team is out of range THEN RETURN false
  IF points is empty THEN RETURN false

  SELECT entity.respawn_point_selector             # a byte in the spawn record
    = 0xFE : RETURN true                           # the record's own coordinates stand
    = 0xFF
    = 0xFD : chosen := a uniformly random point    # "find the best" — never implemented
    otherwise : chosen := that index
                warn if the index is out of range  # and then use it anyway

  entity.position := points[chosen].xyz
  entity.orientation := yaw from points[chosen].w  # pitch and roll are always zero
  RETURN true
```

**Notes** — **Three of the 256 selector values are reserved sentinels and the rest are indices.**
That packing — a selector byte that is either a command or an index — is in the frozen spawn
record format, so a rebuild reading shipped spawn files meets it whether or not it implements
this file.

Two distinct sentinels, `0xFF` and `0xFD`, were both meant to mean "choose well" and both fell
through to a uniform random choice. The difference between them is not recoverable from the
source; most likely one meant "anywhere" and the other "away from enemies", and the distinction
died with the unimplemented chooser.

A respawn point is stored as four numbers — a position and a single angle. Spawning always
levels the entity, which is why a respawn point cannot place a player on a slope facing downhill.

The out-of-range index warns and then proceeds to use the bad index, which would read past the
list. That the whole function is disabled is the only reason it is not a defect.
